<template>
  <div class="col-12 q-pa-md">
    <ChartToolbar @change-chart-type="changeChartType" @export-excel="exportToExcel" @export-png="exportToPng"
      @export-pdf="exportToPdf" @toggle-table="toggleTable" />
    <div v-if="!isTableVisible">
      <div ref="chartContainer"
        style="width: 100%; background-color: white; padding: 20px; border-radius: 10px; position: relative;">
        <div style="width: 100%; height: 400px; position: relative">
          <canvas ref="chartCanvas" style="width: 100%; height: 100%"></canvas>
        </div>
      </div>
    </div>
    <div v-else>
      <q-card>
        <q-table :title="title" :rows="chartData" :columns="tableColumns" row-key="franja_horaria"
          :pagination="{ rowsPerPage: 10 }" />
      </q-card>
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

// Inicializar Quasar
const $q = useQuasar();

// Registrar todas las funcionalidades necesarias de Chart.js
Chart.register(...registerables);

// Props
const props = defineProps({
  title: {
    type: String,
    default: "Movilización de Adultos Mayores por Hora"
  },
  endpoint: {
    type: String,
    default: import.meta.env.VITE_GR_HORASA_URL
  },
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
const chartData = ref([]);
const chartCanvas = ref(null);
const chartContainer = ref(null);
let chartInstance = null;
const currentChartType = ref("line");
const isTableVisible = ref(false);

// Función para generar el timestamp en formato YYYYMMDDHHmmss
const generateTimestamp = () => {
  const now = new Date();
  const year = now.getFullYear();
  const month = String(now.getMonth() + 1).padStart(2, '0');
  const day = String(now.getDate()).padStart(2, '0');
  const hours = String(now.getHours()).padStart(2, '0');
  const minutes = String(now.getMinutes()).padStart(2, '0');
  const seconds = String(now.getSeconds()).padStart(2, '0');
  return `${year}${month}${day}${hours}${minutes}${seconds}`;
};

// Renderizar la gráfica de línea
const renderChart = () => {
  if (!chartCanvas.value) return;

  // Destruir instancia anterior
  if (chartInstance) {
    chartInstance.destroy();
  }

  const ctx = chartCanvas.value.getContext("2d");
  const formattedDate = new Date().toLocaleDateString("es-ES", {
    day: "2-digit",
    month: "2-digit",
    year: "numeric",
    hour: "2-digit",
    minute: "2-digit",
    hour12: false // Esto fuerza el formato de 24 horas
  });

  // Configuración específica para la gráfica de línea
  chartInstance = new Chart(ctx, {
    type: 'line',
    data: {
      labels: chartData.value.map(item => item.franja_horaria),
      datasets: [
        {
          label: 'Cantidad por hora',
          data: chartData.value.map(item => parseInt(item.cantidad)),
          borderColor: '#273984',
          backgroundColor: 'rgba(39, 57, 132, 0.1)',
          borderWidth: 3,
          tension: 0.3,
          fill: true,
          yAxisID: 'y'
        },
        {
          label: 'Acumulado',
          data: chartData.value.map(item => parseInt(item.acumulado)),
          borderColor: '#4CAF50',
          backgroundColor: 'rgba(76, 175, 80, 0.1)',
          borderWidth: 2,
          borderDash: [5, 5],
          tension: 0.1,
          fill: false,
          yAxisID: 'y1'
        }
      ]
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      plugins: {
        title: {
          display: true,
          text: `${props.title}\n${formattedDate}`,
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
    }
  });
};

// Obtener datos para la gráfica
const fetchChartData = async () => {
  try {
    const response = await axios.get(props.endpoint);
    chartData.value = response.data;
    await nextTick();
    renderChart();
  } catch (error) {
    console.error(`Error al obtener los datos de movilización por hora:`, error);
    $q.notify({
      type: 'negative',
      message: 'Error al cargar los datos de movilización',
      position: 'top'
    });
  }
};

// Cambiar tipo de gráfica (aunque por ahora solo mostramos línea)
const changeChartType = (type) => {
  currentChartType.value = type;
  isTableVisible.value = false;
  nextTick(() => renderChart());
};

// Toggle tabla
const toggleTable = () => {
  isTableVisible.value = !isTableVisible.value;
};

// Exportar a Excel
const exportToExcel = () => {
  try {
    const timestamp = generateTimestamp();
    const ws = XLSX.utils.json_to_sheet(chartData.value);
    const wb = XLSX.utils.book_new();
    XLSX.utils.book_append_sheet(wb, ws, 'Datos Movilización');
    XLSX.writeFile(wb, `${timestamp}_movilizacion_por_hora.xlsx`);
    $q.notify({
      type: 'positive',
      message: 'Exportación a Excel exitosa',
      position: 'top',
    });
  } catch (error) {
    console.error("Error exportando a Excel:", error);
    $q.notify({
      type: 'negative',
      message: 'Error al exportar a Excel',
      position: 'top',
    });
  }
};

// Exportar a PNG
const exportToPng = () => {
  if (!chartContainer.value || isTableVisible.value) {
    $q.notify({
      type: 'negative',
      message: 'No se puede exportar la tabla a PNG',
      position: 'top',
    });
    return;
  }

  try {
    html2canvas(chartContainer.value.querySelector("canvas")).then((canvas) => {
      if (!canvas) {
        $q.notify({
          type: 'negative',
          message: 'No se encontró el gráfico para exportar',
          position: 'top',
        });
        return;
      }

      const imgData = canvas.toDataURL("image/png");
      const link = document.createElement("a");
      const timestamp = generateTimestamp();
      link.download = `${timestamp}_movilizacion_por_hora.png`;
      link.href = imgData;
      link.click();
      $q.notify({
        type: 'positive',
        message: 'Exportación a PNG exitosa',
        position: 'top',
      });
    });
  } catch (error) {
    console.error("Error generando imagen:", error);
    $q.notify({
      type: 'negative',
      message: 'Error al exportar a PNG',
      position: 'top',
    });
  }
};

// Exportar a PDF
const exportToPdf = () => {
  if (!chartContainer.value || isTableVisible.value) {
    $q.notify({
      type: 'negative',
      message: 'No se puede exportar la tabla a PDF',
      position: 'top',
    });
    return;
  }

  try {
    html2canvas(chartContainer.value.querySelector("canvas")).then((canvas) => {
      if (!canvas) {
        $q.notify({
          type: 'negative',
          message: 'No se encontró el gráfico para exportar',
          position: 'top',
        });
        return;
      }

      const imgData = canvas.toDataURL("image/png");
      const pdf = new jsPDF('landscape');
      const timestamp = generateTimestamp();

      // Ajustar la imagen al PDF manteniendo la relación de aspecto
      const imgProps = pdf.getImageProperties(imgData);
      const pdfWidth = pdf.internal.pageSize.getWidth() - 20;
      const pdfHeight = (imgProps.height * pdfWidth) / imgProps.width;

      pdf.addImage(imgData, "PNG", 10, 10, pdfWidth, pdfHeight);
      pdf.save(`${timestamp}_movilizacion_por_hora.pdf`);
      $q.notify({
        type: 'positive',
        message: 'Exportación a PDF exitosa',
        position: 'top',
      });
    });
  } catch (error) {
    console.error("Error generando PDF:", error);
    $q.notify({
      type: 'negative',
      message: 'Error al exportar a PDF',
      position: 'top',
    });
  }
};

// Ciclo de vida
onMounted(() => {
  fetchChartData();

  // Escuchar eventos de actualización desde WebSocket
  socket.addEventListener("message", async () => {
    await fetchChartData();
  });
});

onUnmounted(() => {
  if (chartInstance) {
    chartInstance.destroy();
  }
});
</script>

<style scoped>
/* Estilos específicos para este componente */
.q-card {
  width: 100%;
}

canvas {
  width: 100%;
  height: 100%;
  min-height: 400px;
}
</style>
