<template>
  <div class="layout-container">
    <aside class="sidebar">
      <div class="glass-panel">
        <div class="panel-header">
          <h3>Printer & Layout Settings</h3>
        </div>
        <div class="panel-body">
          <div class="row">
            <div class="form-group">
              <label>Unit</label>
              <select v-model="settings.unit" class="form-input">
                <option value="mm">Millimeters (mm)</option>
                <option value="in">Inches (in)</option>
              </select>
            </div>
            <div class="form-group">
              <label>Labels Per Row</label>
              <input type="number" v-model.number="settings.labelsPerRow" class="form-input" min="1" max="10" />
            </div>
          </div>
          <div class="row">
            <div class="form-group">
              <label>Paper Width</label>
              <input type="number" v-model.number="settings.paperWidth" class="form-input" />
            </div>
            <div class="form-group">
              <label>Paper Height (Optional)</label>
              <input type="number" v-model.number="settings.paperHeight" class="form-input" placeholder="Auto" />
            </div>
          </div>
          <div class="row">
            <div class="form-group">
              <label>Label Width (Optional)</label>
              <input type="number" v-model.number="settings.labelWidth" class="form-input" placeholder="Auto" />
            </div>
            <div class="form-group">
              <label>Label Height</label>
              <input type="number" v-model.number="settings.labelHeight" class="form-input" />
            </div>
          </div>
          <div class="form-group">
            <label>Total Labels</label>
            <input type="number" v-model.number="settings.totalLabels" class="form-input" min="1" />
          </div>
        </div>
      </div>

      <div class="glass-panel">
        <div class="panel-header">
          <h3>Label Data</h3>
        </div>
        <div class="panel-body">
          <div class="form-group">
            <label>Item Name</label>
            <input type="text" v-model="labelData.itemName" class="form-input" placeholder="e.g. Premium Widget" />
          </div>
          <div class="form-group">
            <label>Barcode</label>
            <input type="text" v-model="labelData.barcode" class="form-input" placeholder="e.g. 123456789012" />
          </div>
          <div class="row">
            <div class="form-group">
              <label>Exp Date</label>
              <input type="text" v-model="labelData.expDate" class="form-input" placeholder="MM/YYYY" />
            </div>
            <div class="form-group">
              <label>Mfg Date</label>
              <input type="text" v-model="labelData.mfgDate" class="form-input" placeholder="MM/YYYY" />
            </div>
          </div>
          <div class="form-group">
            <label>Lot Number</label>
            <input type="text" v-model="labelData.lotNumber" class="form-input" placeholder="e.g. L-9982" />
          </div>
        </div>
      </div>
    </aside>

    <main class="main-content">
      <div class="preview-header">
        <h2>Live Preview</h2>
        <button class="btn btn-primary" @click="exportPDF" :disabled="isExporting">
          {{ isExporting ? 'Generating PDF...' : 'Export to PDF' }}
        </button>
      </div>
      
      <div class="preview-container glass-panel">
        <div 
          ref="paperRef"
          class="paper-preview" 
          :style="paperStyle"
        >
          <div 
            v-for="i in settings.totalLabels" 
            :key="i"
            class="label-item"
            :style="labelStyle"
          >
            <div class="label-header">{{ labelData.itemName || 'Item Name' }}</div>
            <svg :ref="el => { if(el) barcodeRefs[i-1] = el }"></svg>
            <div class="label-footer">
              <span class="label-footer-item">Exp: {{ labelData.expDate || 'N/A' }}</span>
              <span class="label-footer-item">Lot: {{ labelData.lotNumber || 'N/A' }}</span>
              <span class="label-footer-item">Mfg: {{ labelData.mfgDate || 'N/A' }}</span>
            </div>
          </div>
        </div>
      </div>
    </main>
  </div>
</template>

<script setup>
import { ref, reactive, computed, watch, onMounted, nextTick } from 'vue'
import JsBarcode from 'jsbarcode'
import { jsPDF } from 'jspdf'
import html2canvas from 'html2canvas'

// State
const settings = reactive({
  unit: 'mm',
  labelsPerRow: 2,
  paperWidth: 100, // Total paper width
  paperHeight: null, // Total paper height (auto if null)
  labelWidth: null, // Optional explicit label width
  labelHeight: 25, // Height of a single label
  totalLabels: 10
})

const labelData = reactive({
  itemName: 'Super Widget',
  barcode: '123456789012',
  expDate: '12/2026',
  mfgDate: '01/2026',
  lotNumber: 'L-10293'
})

const barcodeRefs = ref([])
const paperRef = ref(null)
const isExporting = ref(false)

// Conversion helper for CSS
const toCSSUnit = (val) => {
  if (!val) return 'auto'
  return `${val}${settings.unit}`
}

// Computed Styles
const paperStyle = computed(() => {
  return {
    width: toCSSUnit(settings.paperWidth),
    height: toCSSUnit(settings.paperHeight),
    padding: '0',
    display: 'grid',
    gridTemplateColumns: settings.labelWidth 
      ? `repeat(${settings.labelsPerRow}, ${toCSSUnit(settings.labelWidth)})` 
      : `repeat(${settings.labelsPerRow}, 1fr)`,
    justifyContent: 'center',
    alignContent: 'start'
  }
})

const labelStyle = computed(() => {
  return {
    width: '100%',
    height: toCSSUnit(settings.labelHeight)
  }
})

// Barcode rendering logic
const renderBarcodes = () => {
  if (!labelData.barcode) return;
  
  // Need to wait for DOM to update if array size changed
  nextTick(() => {
    barcodeRefs.value.forEach(svgEl => {
      if (svgEl) {
        try {
          JsBarcode(svgEl, labelData.barcode, {
            format: "CODE128",
            displayValue: true,
            fontSize: 12,
            margin: 2,
            height: 30,
            width: 1.2
          })
        } catch(e) {
          console.error("Invalid barcode", e)
        }
      }
    })
  })
}

// Watchers
watch(
  () => [labelData.barcode, settings.totalLabels],
  () => {
    // Reset refs array length
    barcodeRefs.value = new Array(settings.totalLabels).fill(null)
    renderBarcodes()
  },
  { deep: true }
)

onMounted(() => {
  const savedSettings = localStorage.getItem('barcodeSettings')
  if (savedSettings) {
    Object.assign(settings, JSON.parse(savedSettings))
  }
  const savedLabelData = localStorage.getItem('barcodeLabelData')
  if (savedLabelData) {
    Object.assign(labelData, JSON.parse(savedLabelData))
  }
  renderBarcodes()
})

// Save state to local storage automatically
watch(settings, (newVal) => {
  localStorage.setItem('barcodeSettings', JSON.stringify(newVal))
}, { deep: true })

watch(labelData, (newVal) => {
  localStorage.setItem('barcodeLabelData', JSON.stringify(newVal))
}, { deep: true })

// PDF Export Logic
const exportPDF = async () => {
  if (!paperRef.value) return;
  isExporting.value = true;
  
  try {
    // 1. Capture the DOM element with html2canvas
    const canvas = await html2canvas(paperRef.value, {
      scale: 2, // Higher resolution
      useCORS: true,
      logging: false
    });
    
    const imgData = canvas.toDataURL('image/png');
    
    // 2. Calculate PDF dimensions
    // jsPDF uses pt by default, but we can configure it to use our unit (mm or in)
    const pdfUnit = settings.unit === 'in' ? 'in' : 'mm';
    
    // Need to calculate physical size for PDF
    // We assume the paperWidth input corresponds to physical paper size
    const pdfWidth = settings.paperWidth;
    
    // Calculate aspect ratio to determine height if not set
    const imgRatio = canvas.width / canvas.height;
    const pdfHeight = settings.paperHeight || (pdfWidth / imgRatio);
    
    // 3. Create PDF and add image
    const pdf = new jsPDF({
      orientation: pdfWidth > pdfHeight ? 'landscape' : 'portrait',
      unit: pdfUnit,
      format: [pdfWidth, pdfHeight]
    });
    
    pdf.addImage(imgData, 'PNG', 0, 0, pdfWidth, pdfHeight);
    pdf.save('barcode-labels.pdf');
    
  } catch (err) {
    console.error('Error generating PDF:', err);
    alert('Failed to generate PDF. Check console for details.');
  } finally {
    isExporting.value = false;
  }
}
</script>
