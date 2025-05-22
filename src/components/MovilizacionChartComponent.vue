<template>
  <div class="col-12 q-pa-md">
    <!-- Gráfico Adultos Mayores -->
    <div class="q-mb-md">
      <ChartToolbar
        title="Adultos Mayores"
        @change-chart-type="(type) => changeChartType(type, 1)"
        @export-excel="() => exportToExcel(1)"
        @export-png="() => exportToPng(1)"
        @export-pdf="() => exportToPdf(1)"
        @toggle-table="() => toggleTable(1)"
      />
      <div v-if="!isTableVisible1">
        <div
          ref="chartContainer1"
          style="width: 100%; background-color: white; padding: 20px; border-radius: 10px; position: relative;"
        >
          <div style="width: 100%; height: 400px; position: relative">
            <canvas ref="chartCanvas1" style="width: 100%; height: 100%"></canvas>
          </div>
        </div>
      </div>
      <div v-else>
        <q-card>
          <q-table
            title="Datos Adultos Mayores"
            :rows="chartData1"
            :columns="tableColumns"
            row-key="franja_horaria"
            :pagination="{ rowsPerPage: 10 }"
          />
        </q-card>
      </div>
    </div>

    <!-- Gráfico Servidores Públicos -->
    <div>
      <ChartToolbar
        title="Servidores Públicos"
        @change-chart-type="(type) => changeChartType(type, 2)"
        @export-excel="() => exportToExcel(2)"
        @export-png="() => exportToPng(2)"
        @export-pdf="() => exportToPdf(2)"
        @toggle-table="() => toggleTable(2)"
      />
      <div v-if="!isTableVisible2">
        <div
          ref="chartContainer2"
          style="width: 100%; background-color: white; padding: 20px; border-radius: 10px; position: relative;"
        >
          <div style="width: 100%; height: 400px; position: relative">
            <canvas ref="chartCanvas2" style="width: 100%; height: 100%"></canvas>
          </div>
        </div>
      </div>
      <div v-else>
        <q-card>
          <q-table
            title="Datos Servidores Públicos"
            :rows="chartData2"
            :columns="tableColumns"
            row-key="franja_horaria"
            :pagination="{ rowsPerPage: 10 }"
          />
        </q-card>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, nextTick } from "vue";
import axios from "axios";
import { Chart, registerables } from "chart.js";
import "chartjs-plugin-datalabels";
import html2canvas from "html2canvas";
import jsPDF from "jspdf";
import * as XLSX from 'xlsx';
import { useQuasar } from 'quasar';
import ChartToolbar from "src/components/ChartToolbar.vue";
import socket from "src/services/websocket.js";

const $q = useQuasar();
Chart.register(...registerables);

// Props
const props = defineProps({
  tableColumns: {
    type: Array,
    default: () => [
      {
        name: 'franja_horaria',
        required: true,
        label: 'Franja Horaria',
        align: 'left',
        field: 'franja_horaria',
        sortable: true
      },
      {
        name: 'cantidad',
        label: 'Cantidad',
        align: 'center',
        field: 'cantidad',
        sortable: true
      },
      {
        name: 'acumulado',
        label: 'Acumulado',
        align: 'center',
        field: 'acumulado',
        sortable: true
      }
    ]
  }
});

// Variables reactivas
const chartData1 = ref([]); // Adultos mayores
const chartData2 = ref([]); // Servidores públicos
const chartCanvas1 = ref(null);
const chartCanvas2 = ref(null);
const chartContainer1 = ref(null);
const chartContainer2 = ref(null);
let chartInstance1 = null;
let chartInstance2 = null;
const currentChartType1 = ref("line");
const currentChartType2 = ref("line");
const isTableVisible1 = ref(false);
const isTableVisible2 = ref(false);

// Función para generar el timestamp
const generateTimestamp = () => {
  const now = new Date();
  return now.toISOString().replace(/[:.]/g, '-').slice(0, 19);
};

// Formatear fecha y hora
const formatDateTime = () => {
  return new Date().toLocaleDateString("es-ES", {
    day: "2-digit",
    month: "2-digit",
    year: "numeric",
    hour: "2-digit",
    minute: "2-digit",
    hour12: false
  });
};

// Configuración común para gráficos
const getChartOptions = (title) => {
  return {
    responsive: true,
    maintainAspectRatio: false,
    plugins: {
      title: {
        display: true,
        text: `${title}\n${formatDateTime()}`,
        font: {
          size: 16
        }
      },
      tooltip: {
        mode: 'index',
        intersect: false
      },
      legend: {
        position: 'top'
      }
    },
    scales: {
      x: {
        title: {
          display: true,
          text: 'Franja Horaria'
        },
        grid: {
          display: false
        }
      },
      y: {
        type: 'linear',
        display: true,
        position: 'left',
        title: {
          display: true,
          text: 'Cantidad por hora'
        },
        min: 0
      },
      y1: {
        type: 'linear',
        display: true,
        position: 'right',
        title: {
          display: true,
          text: 'Acumulado'
        },
        min: 0,
        grid: {
          drawOnChartArea: false
        }
      }
    },
    interaction: {
      mode: 'nearest',
      axis: 'x',
      intersect: false
    }
  };
};

// Renderizar gráfica Adultos Mayores
const renderChart1 = () => {
  if (!chartCanvas1.value) return;

  if (chartInstance1) {
    chartInstance1.destroy();
  }

  const ctx = chartCanvas1.value.getContext("2d");

  chartInstance1 = new Chart(ctx, {
    type: currentChartType1.value,
    data: {
      labels: chartData1.value.map(item => item.franja_horaria),
      datasets: [
        {
          label: 'Cantidad por hora',
          data: chartData1.value.map(item => parseInt(item.cantidad)),
          borderColor: '#273984', // Azul oscuro
          backgroundColor: 'rgba(39, 57, 132, 0.1)',
          borderWidth: 3,
          tension: 0.3,
          fill: true,
          yAxisID: 'y'
        },
        {
          label: 'Acumulado',
          data: chartData1.value.map(item => parseInt(item.acumulado)),
          borderColor: '#4CAF50', // Verde
          backgroundColor: 'rgba(76, 175, 80, 0.1)',
          borderWidth: 2,
          borderDash: [5, 5],
          tension: 0.1,
          fill: false,
          yAxisID: 'y1'
        }
      ]
    },
    options: getChartOptions("Movilización de Adultos Mayores por Hora")
  });
};

// Renderizar gráfica Servidores Públicos
const renderChart2 = () => {
  if (!chartCanvas2.value) return;

  if (chartInstance2) {
    chartInstance2.destroy();
  }

  const ctx = chartCanvas2.value.getContext("2d");

  chartInstance2 = new Chart(ctx, {
    type: currentChartType2.value,
    data: {
      labels: chartData2.value.map(item => item.franja_horaria),
      datasets: [
        {
          label: 'Cantidad por hora',
          data: chartData2.value.map(item => parseInt(item.cantidad)),
          borderColor: '#8E44AD', // Morado
          backgroundColor: 'rgba(142, 68, 173, 0.1)',
          borderWidth: 3,
          tension: 0.3,
          fill: true,
          yAxisID: 'y'
        },
        {
          label: 'Acumulado',
          data: chartData2.value.map(item => parseInt(item.acumulado)),
          borderColor: '#E74C3C', // Rojo
          backgroundColor: 'rgba(231, 76, 60, 0.1)',
          borderWidth: 2,
          borderDash: [5, 5],
          tension: 0.1,
          fill: false,
          yAxisID: 'y1'
        }
      ]
    },
    options: getChartOptions("Movilización de Servidores Públicos por Hora")
  });
};

// Obtener datos para ambas gráficas
const fetchChartData = async () => {
  try {
    // Adultos Mayores
    const response1 = await axios.get(import.meta.env.VITE_GR_HORASA_URL);
    chartData1.value = response1.data;

    // Servidores Públicos
    const response2 = await axios.get(import.meta.env.VITE_GR_HORASS_URL);
    chartData2.value = response2.data;

    await nextTick();
    renderChart1();
    renderChart2();
  } catch (error) {
    console.error("Error al obtener los datos:", error);
    $q.notify({
      type: 'negative',
      message: 'Error al cargar los datos',
      position: 'top'
    });
  }
};

// Cambiar tipo de gráfica (1 para Adultos Mayores, 2 para Servidores Públicos)
const changeChartType = (type, chartNumber) => {
  if (chartNumber === 1) {
    currentChartType1.value = type;
    isTableVisible1.value = false;
    nextTick(() => renderChart1());
  } else {
    currentChartType2.value = type;
    isTableVisible2.value = false;
    nextTick(() => renderChart2());
  }
};

// Toggle tabla (1 para Adultos Mayores, 2 para Servidores Públicos)
const toggleTable = (chartNumber) => {
  if (chartNumber === 1) {
    isTableVisible1.value = !isTableVisible1.value;
  } else {
    isTableVisible2.value = !isTableVisible2.value;
  }
};

// Exportar a Excel (1 para Adultos Mayores, 2 para Servidores Públicos)
const exportToExcel = (chartNumber) => {
  try {
    const timestamp = generateTimestamp();
    const data = chartNumber === 1 ? chartData1.value : chartData2.value;
    const title = chartNumber === 1 ? 'adultos_mayores' : 'servidores_publicos';

    const ws = XLSX.utils.json_to_sheet(data);
    const wb = XLSX.utils.book_new();
    XLSX.utils.book_append_sheet(wb, ws, 'Datos');
    XLSX.writeFile(wb, `${timestamp}_${title}.xlsx`);

    $q.notify({
      type: 'positive',
      message: `Exportación a Excel exitosa (${chartNumber === 1 ? 'Adultos Mayores' : 'Servidores Públicos'})`,
      position: 'top',
    });
  } catch (error) {
    console.error("Error exportando a Excel:", error);
    $q.notify({
      type: 'negative',
      message: `Error al exportar a Excel (${chartNumber === 1 ? 'Adultos Mayores' : 'Servidores Públicos'})`,
      position: 'top',
    });
  }
};

// Exportar a PNG (1 para Adultos Mayores, 2 para Servidores Públicos)
const exportToPng = (chartNumber) => {
  const container = chartNumber === 1 ? chartContainer1.value : chartContainer2.value;
  const isTableVisible = chartNumber === 1 ? isTableVisible1.value : isTableVisible2.value;
  const title = chartNumber === 1 ? 'adultos_mayores' : 'servidores_publicos';

  if (!container || isTableVisible) {
    $q.notify({
      type: 'negative',
      message: `No se puede exportar la tabla a PNG (${chartNumber === 1 ? 'Adultos Mayores' : 'Servidores Públicos'})`,
      position: 'top',
    });
    return;
  }

  try {
    html2canvas(container.querySelector("canvas")).then((canvas) => {
      const imgData = canvas.toDataURL("image/png");
      const link = document.createElement("a");
      link.download = `${generateTimestamp()}_${title}.png`;
      link.href = imgData;
      link.click();

      $q.notify({
        type: 'positive',
        message: `Exportación a PNG exitosa (${chartNumber === 1 ? 'Adultos Mayores' : 'Servidores Públicos'})`,
        position: 'top',
      });
    });
  } catch (error) {
    console.error("Error generando imagen:", error);
    $q.notify({
      type: 'negative',
      message: `Error al exportar a PNG (${chartNumber === 1 ? 'Adultos Mayores' : 'Servidores Públicos'})`,
      position: 'top',
    });
  }
};

// Exportar a PDF (1 para Adultos Mayores, 2 para Servidores Públicos)
const exportToPdf = (chartNumber) => {
  const container = chartNumber === 1 ? chartContainer1.value : chartContainer2.value;
  const isTableVisible = chartNumber === 1 ? isTableVisible1.value : isTableVisible2.value;
  const title = chartNumber === 1 ? 'Adultos Mayores' : 'Servidores Públicos';

  if (!container || isTableVisible) {
    $q.notify({
      type: 'negative',
      message: `No se puede exportar la tabla a PDF (${title})`,
      position: 'top',
    });
    return;
  }

  try {
    html2canvas(container.querySelector("canvas")).then((canvas) => {
      const imgData = canvas.toDataURL("image/png");
      const pdf = new jsPDF('landscape');
      const timestamp = generateTimestamp();

      const imgProps = pdf.getImageProperties(imgData);
      const pdfWidth = pdf.internal.pageSize.getWidth() - 20;
      const pdfHeight = (imgProps.height * pdfWidth) / imgProps.width;

      pdf.addImage(imgData, "PNG", 10, 10, pdfWidth, pdfHeight);
      pdf.save(`${timestamp}_movilizacion_${chartNumber === 1 ? 'adultos_mayores' : 'servidores_publicos'}.pdf`);

      $q.notify({
        type: 'positive',
        message: `Exportación a PDF exitosa (${title})`,
        position: 'top',
      });
    });
  } catch (error) {
    console.error("Error generando PDF:", error);
    $q.notify({
      type: 'negative',
      message: `Error al exportar a PDF (${title})`,
      position: 'top',
    });
  }
};

// Ciclo de vida
onMounted(() => {
  fetchChartData();

  socket.addEventListener("message", async () => {
    await fetchChartData();
  });
});

onUnmounted(() => {
  if (chartInstance1) chartInstance1.destroy();
  if (chartInstance2) chartInstance2.destroy();
});
</script>

<style scoped>
.q-card {
  width: 100%;
}

canvas {
  width: 100%;
  height: 100%;
  min-height: 400px;
}

.q-mb-md {
  margin-bottom: 16px;
}
</style>
