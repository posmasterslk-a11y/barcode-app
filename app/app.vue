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
          <h3>Typography (Font Sizes)</h3>
        </div>
        <div class="panel-body">
          <div class="row">
            <div class="form-group">
              <label>Shop Name</label>
              <input type="number" v-model.number="settings.fontSizeShop" class="form-input" min="5" max="50" />
            </div>
            <div class="form-group">
              <label>Item Name</label>
              <input type="number" v-model.number="settings.fontSizeItem" class="form-input" min="5" max="50" />
            </div>
          </div>
          <div class="row">
            <div class="form-group">
              <label>Barcode Text</label>
              <input type="number" v-model.number="settings.fontSizeBarcode" class="form-input" min="5" max="50" />
            </div>
            <div class="form-group">
              <label>Footer</label>
              <input type="number" v-model.number="settings.fontSizeFooter" class="form-input" min="5" max="50" />
            </div>
          </div>
        </div>
      </div>

      <div class="glass-panel">
        <div class="panel-header">
          <h3>Label Data</h3>
        </div>
        <div class="panel-body">
          <div class="form-group">
            <label>Shop Name</label>
            <input type="text" v-model="labelData.shopName" class="form-input" placeholder="e.g. My Super Shop" />
          </div>
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
              <input type="date" v-model="labelData.expDate" class="form-input" />
            </div>
            <div class="form-group">
              <label>Mfg Date</label>
              <input type="date" v-model="labelData.mfgDate" class="form-input" />
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
            <div class="label-shop-name" :style="{ fontSize: settings.fontSizeShop + 'px', fontWeight: 'bold', marginBottom: '2px' }">{{ labelData.shopName || 'Shop Name' }}</div>
            <div class="label-header" :style="{ fontSize: settings.fontSizeItem + 'px' }">{{ labelData.itemName || 'Item Name' }}</div>
            <img :ref="el => { if(el) barcodeRefs[i-1] = el }" class="label-barcode-img" />
            <div class="label-footer" :style="{ fontSize: settings.fontSizeFooter + 'px' }">
              <span class="label-footer-item">Exp: {{ labelData.expDate || 'N/A' }}</span>
              <span class="label-footer-item">Lot: {{ labelData.lotNumber || 'N/A' }}</span>
              <span class="label-footer-item">Mfg: {{ labelData.mfgDate || 'N/A' }}</span>
            </div>
          </div>
        </div>
      </div>
      <footer style="text-align: center; margin-top: 1.5rem; color: var(--text-secondary); font-size: 0.9rem;">
        Powered By POS Masters : 070 296 7270
      </footer>
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
  totalLabels: 10,
  fontSizeShop: 11,
  fontSizeItem: 14,
  fontSizeBarcode: 12,
  fontSizeFooter: 10
})

const labelData = reactive({
  shopName: 'My Super Shop',
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
            fontSize: settings.fontSizeBarcode,
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
  () => [labelData.barcode, settings.totalLabels, settings.fontSizeBarcode],
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
  
  // Open the new tab immediately to bypass popup blockers
  const newTab = window.open('', '_blank');
  if (newTab) {
    newTab.document.write('<div style="font-family: sans-serif; text-align: center; margin-top: 50px;"><h2>Generating your PDF...</h2><p>Please wait a moment.</p></div>');
  }

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
    const pdfUnit = settings.unit === 'in' ? 'in' : 'mm';
    const pWidth = parseFloat(settings.paperWidth) || 100;
    const imgRatio = canvas.width / canvas.height;
    const pHeight = settings.paperHeight ? parseFloat(settings.paperHeight) : (pWidth / imgRatio);
    
    // 3. Create PDF and add image
    const pdf = new jsPDF({
      orientation: pWidth > pHeight ? 'landscape' : 'portrait',
      unit: pdfUnit,
      format: [pWidth, pHeight]
    });
    
    pdf.addImage(imgData, 'PNG', 0, 0, pWidth, pHeight);
    
    // Pass the blob to the existing new tab
    const pdfBlob = pdf.output('blob');
    const pdfBlobUrl = URL.createObjectURL(pdfBlob);
    
    if (newTab) {
      newTab.location.href = pdfBlobUrl;
    } else {
      // Fallback if popup blocker caught the synchronous open (rare)
      window.open(pdfBlobUrl, '_blank');
    }
    
  } catch (err) {
    console.error('Error generating PDF:', err);
    if (newTab) newTab.close();
    alert('Failed to generate PDF. Check console for details.');
  } finally {
    isExporting.value = false;
  }
}
</script>
