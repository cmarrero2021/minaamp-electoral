<template>
  <div class="q-pa-md">
    <h4 class="q-mb-md">SERVIDORES</h4>

    <q-table title="Lista de Revistas" :rows="filteredJournals" :columns="columns" :rows-per-page-options="[10, 20, 50]"
      row-key="id" :pagination="pagination" :loading="loading" virtual-scroll class="responsive-table">
      <!-- Búsqueda general y botón borrar filtros -->
      <template v-slot:top>
        <!-- Primera fila: Búsqueda general y botón borrar filtros -->
        <div class="full-width row wrap items-center q-mb-md">
          <!-- Búsqueda general -->
          <div class="col-xs-10 col-sm-5 q-pr-xs">
            <q-input outlined dense debounce="300" v-model="searchQuery" label="Búsqueda general"
              placeholder="Buscar en todos los campos">
              <template v-slot:append>
                <q-icon v-if="searchQuery" name="clear" @click.stop="clearSearch" class="cursor-pointer" size="sm" />
              </template>
            </q-input>
          </div>
          <!-- Botón "Borrar todos los filtros" -->
          <div class="col-xs-2 col-sm-1">
            <q-btn icon="fas fa-trash" title="Borrar todos los filtros" @click="clearAllFilters" color="negative" flat
              size="sm" class="full-width" />
          </div>
        </div>

        <!-- Segunda fila: Filtros para las columnas -->
        <div class="full-width row wrap  items-center content-center q-mb-md">
          <div class="col-xs-12 col-sm-6 col-md-3 q-pa-sm" v-for="col in columns" :key="col.name">
            <div v-if="col.filterable">
              <div v-if="col.type === 'select'">
                <q-select :model-value="filters[col.name]" @update:model-value="val => updateFilter(col.name, val)"
                  :options="getOptions(col.name)" :label="col.label" multiple outlined dense use-chips clearable>
                  <template v-slot:append>
                    <q-icon v-if="filters[col.name] && filters[col.name].length > 0" name="clear"
                      @click.stop="clearFilter(col.name)" class="cursor-pointer" size="sm" />
                  </template>
                </q-select>
              </div>
              <div v-else>
                <q-input :model-value="filters[col.name]" @update:model-value="val => updateFilter(col.name, val)"
                  :label="col.label" placeholder="Filtrar" outlined dense debounce="300">
                  <template v-slot:append>
                    <q-icon v-if="filters[col.name]" name="clear" @click.stop="clearFilter(col.name)"
                      class="cursor-pointer" size="sm" />
                  </template>
                </q-input>
              </div>
            </div>
          </div>
        </div>
        <div class="col-xs-2 col-sm-1">
          <q-btn icon="add" title="Agregar nueva revista" @click="openNewModal" color="positive" size="sm"
            class="full-width" />
        </div>
      </template>

      <!-- Botones de acción en cada fila -->
      <template v-slot:body-cell-actions="props">
        <q-td>
          <div class="row items-center">
            <!-- Botón Editar -->
            <q-btn icon="edit" color="primary" title="Editar servidor" size="xs" @click.stop="openEditModal(props.row)"
              class="q-mr-xs" />

            <!-- Botón Ver -->
            <!-- <q-btn icon="visibility" color="positive" title="Ver servidor" size="xs" class="q-mr-xs" /> -->

            <!-- Botón Borrar -->
            <q-btn icon="delete" color="negative" title="Eliminar Servidor" size="xs" class="q-mr-xs" />
          </div>
        </q-td>
      </template>

      <!-- Estado de carga -->
      <template v-slot:loading>
        <q-inner-loading showing color="primary" />
      </template>
    </q-table>

    <!-- Modal de Edición -->
    <q-dialog v-model="editDialog" persistent>
      <q-card style="width: 700px; max-width: 80vw;">
        <q-card-section>
          <div class="text-h6">{{ isEditing ? 'Editar Servidor' : 'Cargar Servidor' }}</div>
        </q-card-section>

        <q-card-section>
          <q-form @submit="saveChanges" class="q-gutter-md">
            <div class="row q-col-gutter-md">
              <!-- Campos del formulario -->
              <div class="col-12 col-md-6">
                <q-input v-model="editForm.id" label="ID" readonly filled />
              </div>
              <div class="col-12 col-md-6">
                <q-input v-model="editForm.cedula" label="Cédula" type="number" filled />
              </div>
              <div class="col-12 col-md-6">
                <q-input :model-value="editForm.nombres"
                  @update:model-value="val => editForm.nombres = val.toUpperCase()" label="Nombres" filled />
              </div>
              <div class="col-12 col-md-6">
                <q-input v-model="editForm.hora_voto" label="Votó" type="time" filled />
              </div>
              <div class="col-12 col-md-6">
                <q-select v-model="editForm.institucion" :options="optionsu.institucion" label="Institución" filled
                  option-label="label" option-value="value" />
              </div>
              <div class="col-12 col-md-6">
                <q-select v-model="editForm.sede" :options="optionsu.sede" label="Sede" filled option-label="label"
                  option-value="value" />
              </div>
              <div class="col-12 col-md-6">
                <q-select v-model="editForm.area" :options="optionsu.area" label="Área" filled option-label="label"
                  option-value="value" />
              </div>
              <div class="col-12 col-md-6">
                <q-select v-model="editForm.estado" :options="optionsu.estado" label="Estado" filled
                  option-label="label" option-value="value" />
              </div>
              <div class="col-12">
                <q-input :model-value="editForm.observaciones"
                  @update:model-value="val => editForm.observaciones = val.toUpperCase()" label="Observaciones"
                  type="textarea" filled />
              </div>
            </div>
            <div class="row justify-end">
              <q-btn icon="cancel" color="negative" type="reset" @click="closeEditModal" />
              <q-btn icon="save" color="primary" type="submit" class="q-ml-sm" />
            </div>
          </q-form>
        </q-card-section>
      </q-card>
    </q-dialog>
  </div>
</template>

<script setup>
import { ref, onMounted, computed, watch } from 'vue';
import { LocalStorage, Notify } from 'quasar'
import axios from 'axios';
// Definición de columnas para la tabla
const columns = [
  { name: 'actions', label: 'Acciones', align: 'left' },
  { name: 'cedula', label: 'Cédula', field: 'cedula', sortable: true, filterable: true, align: 'left', type: 'text' },
  { name: 'nombres', label: 'Nombre', field: 'nombres', sortable: true, filterable: true, align: 'left', type: 'text' },
  { name: 'hora_voto', label: 'Votó', field: 'hora_voto', sortable: true, filterable: false, align: 'left', type: 'text' },
  { name: 'institucion', label: 'Institución', field: 'institucion', sortable: true, filterable: true, align: 'left', type: 'select' },
  { name: 'sedes', label: 'Sede', field: 'sede', sortable: true, filterable: true, align: 'left', type: 'select' },
  { name: 'areas', label: 'Área', field: 'area', sortable: true, filterable: true, align: 'left', type: 'select' },
  { name: 'estado', label: 'Estado', field: 'estado', sortable: true, filterable: true, align: 'left', type: 'select' },
  { name: 'observaciones', label: 'Observaciones', field: 'observaciones', sortable: false, filterable: false, align: 'left', type: 'text' },
];

// URLs de los endpoints
const apiURL = import.meta.env.VITE_API_URL;
const estadosURL = import.meta.env.VITE_ESTADOR_BASE_URL;
const estadosLsURL = import.meta.env.VITE_LS_ESTADOS_URL;
const servidoresURL = import.meta.env.VITE_LS_SERVERS_URL
const institucionesURL = import.meta.env.VITE_LS_INSTITUTIONS_URL;
const sedesURL = import.meta.env.VITE_LS_SEDES_URL;
const areasURL = import.meta.env.VITE_LS_AREAS_URL;
const servidorDetailURL = import.meta.env.VITE_BC_SERVER_URL;
const updateServerURL = import.meta.env.VITE_UP_SERVER_URL;
const insertServerURL = import.meta.env.VITE_IN_SERVER_URL;


// Estado de la aplicación
const journals = ref([]);
const loading = ref(true);
const pagination = ref({
  sortBy: 'desc',
  descending: false,
  page: 1,
  rowsPerPage: 10
});

// Búsqueda general
const searchQuery = ref('');

// Filtros para las columnas
const filters = ref({
  areas: null,
  institucion: null,
  sede: null,
  estado: null,
  indice: null
});

// Opciones para los select
const options = ref({
  areas: [],
  institucion: [],
  sede: [],
  estado: [],
});
const optionsu = ref({
  areas: [],
  institucion: [],
  sede: [],
  estado: [],
});


// Estado del modal de edición
const editDialog = ref(false);
const editForm = ref({});

// Función para obtener los datos de las revistas
const fetchJournals = async () => {
  try {
    const response = await axios.get(servidoresURL);
    journals.value = response.data;
  } catch (error) {
    console.error('Error al obtener las revistas:', error);
  } finally {
    loading.value = false;
  }
};

// Función para obtener las opciones de los filtros
const fetchOptions = async () => {
  try {
    const areasResponse = await axios.get(areasURL);
    options.value.areas = areasResponse.data.map(item => item.area);
    const areasResponseU = await axios.get(areasURL);
    optionsu.value.area = areasResponseU.data.map(item => ({
      label: item.area,
      value: item.area_id
    }));

    // Obtener instituciones
    const institucionsResponse = await axios.get(institucionesURL);
    options.value.institucion = institucionsResponse.data.map(item => item.institucion);
    const institucionsResponseU = await axios.get(institucionesURL);
    optionsu.value.institucion = institucionsResponseU.data.map(item => ({
      label: item.institucion,
      value: item.id
    }));
    // Obtener sedes
    const sedesResponse = await axios.get(sedesURL);
    options.value.sedes = sedesResponse.data.map(item => item.sede);
    const sedesResponseU = await axios.get(sedesURL);
    optionsu.value.sede = sedesResponseU.data.map(item => ({
      label: item.sede,
      value: item.id
    }));

    // Obtener estados
    const estadosResponse = await axios.get(estadosURL);
    options.value.estado = estadosResponse.data.map(item => item.estado);
    const estadosResponseU = await axios.get(estadosLsURL);
    optionsu.value.estado = estadosResponseU.data.map(item => ({
      label: item.estado,
      value: item.id
    }));

  } catch (error) {
    console.error('Error al obtener las opciones de los filtros:', error);
  }
};

// Actualizar los filtros
const updateFilter = (key, value) => {
  filters.value[key] = value;
};

// Borrar un filtro específico
const clearFilter = (filterName) => {
  filters.value[filterName] = null;
};

// Borrar todos los filtros
const clearAllFilters = () => {
  for (const filter in filters.value) {
    filters.value[filter] = null;
  }
  searchQuery.value = '';
};

// Borrar la búsqueda general
const clearSearch = () => {
  searchQuery.value = '';
};

// Obtener las opciones para un filtro select
const getOptions = (filterName) => {
  return options.value[filterName] || [];
};

// Calcular la lista de revistas filtradas
const filteredJournals = computed(() => {
  // Primero filtrar por búsqueda general
  const searchValue = searchQuery.value.toLowerCase();
  let searchedJournals = journals.value.filter(journal => {
    // Buscar en todos los campos de las columnas filtrables
    return columns
      .filter(col => col.filterable)
      .some(col => {
        const journalValue = journal[col.field]?.toString()?.toLowerCase() || '';
        return journalValue.includes(searchValue);
      });
  });

  // Luego aplicar los filtros por columna
  return searchedJournals.filter(journal => {
    return columns
      .filter(col => col.filterable)
      .every(col => {
        // Para filtros de tipo select
        if (col.type === 'select' && filters.value[col.name] && filters.value[col.name].length > 0) {
          return filters.value[col.name].includes(journal[col.field]);
        }

        // Para otros tipos de filtros
        const filterValue = filters.value[col.name]?.toLowerCase() || '';
        const journalValue = journal[col.field]?.toString()?.toLowerCase() || '';
        return journalValue.includes(filterValue);
      });
  });
});
const isEditing = ref(false);
// Función para abrir el modal de creación!
const openNewModal = () => {
  editForm.value = {
    // Inicializa todos los campos necesarios
    id: null,
    area_id: null,
    institucion_id: null,
    sede_id: null,
    cedula: null,
    nombres: null,
    hora_voto: null,
    observaciones: null,
    // revista: '',
    // areas: null,
    // indice: null,
    // idioma: null,
    // correo_revista: '',
    // editorial: null,
    // periodicidad: null,
    // formato: null,
    // estado: null,
    // ciudad: '',
    // nombres_editor: '',
    // apellidos_editor: '',
    // correo_editor: '',
    // deposito_legal_impreso: '',
    // deposito_legal_digital: '',
    // issn_impreso: '',
    // issn_digital: '',
    // url: '',
    // anio_inicial: '',
    // direccion: '',
    // telefono: '',
    // resumen: '',
    // portada: null
  };
  editDialog.value = true;
  isEditing.value = false;
};

// Función para abrir el modal de edición
const openEditModal = async (journal) => {
  try {
    const response = await axios.get(`${servidorDetailURL}${journal.cedula}`);
    editForm.value = { ...response.data };
    editDialog.value = true;
    isEditing.value = true;
  } catch (error) {
    console.error('Error al obtener los datos del servidor:', error);
  }
};
////////////////////////
// Función para cerrar el modal de edición
const closeEditModal = () => {
  editDialog.value = false;
};

const saveChanges = async () => {
  try {
    const servidorData = {
      area_id: isEditing.value
        ? editForm.value.area_id
        : editForm.value.area?.value,

      institucion_id: isEditing.value
        ? editForm.value.institucion_id
        : editForm.value.institucion?.value,

      sede_id: isEditing.value
        ? editForm.value.sede_id
        : editForm.value.sede?.value,

      cedula: editForm.value.cedula,
      nombres: editForm.value.nombres?.toUpperCase() ?? '',
      hora_voto: editForm.value.hora_voto,
      observaciones: editForm.value.observaciones?.toUpperCase() ?? '',
    };
    if (isEditing.value) {
      // Modo edición
      await axios.patch(`${updateServerURL}${editForm.value.cedula}`, servidorData);
      Notify.create({
        type: 'positive',
        message: 'Los cambios se han guardado correctamente.'
      });
    } else {
      // Modo creación
      await axios.post(insertServerURL, servidorData);
      Notify.create({
        type: 'positive',
        message: 'La revista se ha creado correctamente.'
      });
    }

    // Actualizar la lista de revistas
    await fetchJournals();
    closeEditModal();
  } catch (error) {
    const cedula = servidorData.cedula;
    const mensaje = error.response.status === 409 ? `La cédula ${cedula} ya se encuentra registrada.` : `Ha ocurrido un error ${error} y no se han guardar los cambios.`;
    Notify.create({
      type: 'negative',
      message: mensaje
    });
  }
};


////////////////
// Observar cambios en la paginación
watch(pagination, () => {

}, { deep: true });

// Obtener los datos al montar el componente
onMounted(async () => {
  if (!LocalStorage.getItem('token')) {
    router.push('/login')
  }
  await fetchOptions();
  fetchJournals();
});
</script>

<style scoped>
.responsive-table {
  max-width: 100%;
  overflow-x: auto;
}

@media (max-width: 768px) {
  .responsive-table {
    overflow-x: scroll;
  }
}
</style>
