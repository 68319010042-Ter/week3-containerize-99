เพื่อเพิ่มประสิทธิภาพและความน่าใช้งานให้กับโค้ดระบบจัดการคลังสินค้า Vue 3 (Composition API) ชุดนี้ ผมได้ทำการเพิ่ม **5 ฟังก์ชันใหม่** ที่ช่วยให้ระบบสมบูรณ์แบบและตอบโจทย์การใช้งานจริงมากขึ้น ดังนี้ครับ:

1. **ระบบคัดกรองสินค้าตามระดับสต็อก (Stock Status Filter):** เพิ่ม `select` ใน Toolbar เพื่อให้ผู้ใช้สามารถเลือกดูเฉพาะสินค้าที่ "หมด", "ใกล้หมด", หรือ "มีเพียงพอ" ได้ทันที
2. **ระบบจัดเรียงลำดับสินค้า (Sort By):** เพิ่มตัวเลือกให้สามารถเรียงลำดับสินค้าตาม "ชื่อ (A-Z)", "ราคา (ต่ำ-สูง / สูง-ต่ำ)", หรือ "จำนวนสต็อก" เพื่อให้หาข้อมูลได้ง่ายขึ้น
3. **ระบบดึงข้อมูลจำลองหาก API พัง (Mock Data Fallback):** ปรับปรุงฟังก์ชัน `fetchProducts` ให้ดึงข้อมูลจำลอง (Mock Data) มาแสดงผลทันทีหากเชื่อมต่อ API หลักไม่ได้ ช่วยให้หน้าเว็บไม่ว่างเปล่าและทดสอบระบบได้ง่ายขึ้น
4. **ปุ่มทางลัดเพิ่ม/ลดสต็อกด่วน (Quick Stock Adjustment):** เพิ่มปุ่ม `+` และ `-` บนการ์ดสินค้าแต่ละชิ้น เพื่อให้กดปรับจำนวนสต็อกเพิ่มลดได้ทันที 1 ชิ้น โดยไม่ต้องกดเข้าไปเปิด Modal แก้ไขให้เสียเวลา
5. **ฟังก์ชันคำนวณราคาเฉลี่ยต่อชิ้น (Average Price Stat Card):** ปรับปรุงระบบ Dashboard ให้คำนวณและแสดงผล "ราคาเฉลี่ยต่อรายการสินค้า" เพิ่มขึ้นมาอีก 1 ตัววัด

---

### โค้ดที่ได้รับการอัปเดตแล้ว

คุณสามารถคัดลอกโค้ดทั้งหมดนี้ไปวางทับไฟล์เดิมเพื่อใช้งานได้เลยครับ:

```vue
<script setup>
import { ref, computed, onMounted } from 'vue'

// ── State ────────────────────────────────────────────────────────
const products      = ref([])       // รายการสินค้าทั้งหมดจาก API
const loading       = ref(true)     // แสดง loading spinner
const search        = ref('')       // ข้อความค้นหา
const catFilter     = ref('')       // category ที่กำลัง filter
const stockFilter   = ref('')       // [NEW] คัดกรองตามระดับสต็อก ('', 'out', 'low', 'mid', 'high')
const sortBy        = ref('name-asc') // [NEW] การจัดเรียงลำดับสินค้า
const showModal     = ref(false)    // แสดง/ซ่อน modal เพิ่ม/แก้ไข
const editingId     = ref(null)     // null = กำลัง add, มีค่า = กำลัง edit
const confirmDelete = ref(null)     // เก็บ product ที่จะลบ (ใช้ confirm dialog)

// ข้อมูลในฟอร์ม modal
const form = ref({ name: '', category: '', price: '', stock: 0, description: '' })

let debounceTimer = null            // สำหรับ debounce search input

// ── Computed ─────────────────────────────────────────────────────

// กรองสินค้าตาม search + catFilter + stockFilter และจัดเรียงด้วย sortBy
const filtered = computed(() => {
  const s = search.value.toLowerCase()
  
  // 1. กรองข้อมูล (Filtering)
  let result = products.value.filter(p => {
    // ตรงกับ search text?
    const matchS = !s ||
      p.name.toLowerCase().includes(s) ||
      (p.description || '').toLowerCase().includes(s)
      
    // ตรงกับ category filter?
    const matchC = !catFilter.value || p.category === catFilter.value
    
    // [NEW] ตรงกับ stock status filter?
    let matchST = true
    if (stockFilter.value) {
      const currentStatus = stockClass(p.stock)
      matchST = currentStatus === stockFilter.value
    }
    
    return matchS && matchC && matchST
  })

  // [NEW] 2. จัดเรียงข้อมูล (Sorting)
  return result.sort((a, b) => {
    if (sortBy.value === 'name-asc')   return a.name.localeCompare(b.name)
    if (sortBy.value === 'name-desc')  return b.name.localeCompare(a.name)
    if (sortBy.value === 'price-asc')  return parseFloat(a.price) - parseFloat(b.price)
    if (sortBy.value === 'price-desc') return parseFloat(b.price) - parseFloat(a.price)
    if (sortBy.value === 'stock-asc')  return a.stock - b.stock
    if (sortBy.value === 'stock-desc') return b.stock - a.stock
    return 0
  })
})

// สร้าง list ของ categories จาก products ที่มีอยู่ (unique + sort)
const categories = computed(() =>
  [...new Set(products.value.map(p => p.category))].sort()
)

// คำนวณ 5 ตัวเลข dashboard stats [NEW: เพิ่ม avgPrice]
const stats = computed(() => {
  const total = products.value.length
  const totalValue = products.value.reduce((s, p) => s + parseFloat(p.price) * p.stock, 0)
  const totalItems = products.value.reduce((s, p) => s + p.stock, 0)
  const avgPrice = total > 0 ? products.value.reduce((s, p) => s + parseFloat(p.price), 0) / total : 0

  return {
    total,
    lowStock:   products.value.filter(p => p.stock < 10).length,
    totalItems,
    totalValue,
    avgPrice // [NEW] ราคาเฉลี่ยต่อรายการสินค้า
  }
})

// ── Methods ───────────────────────────────────────────────────────

// debounce: รอ 280ms หลัง user หยุดพิมพ์แล้วค่อย fetch
function debounceFetch() {
  clearTimeout(debounceTimer)
  debounceTimer = setTimeout(fetchProducts, 280)
}

// ดึงสินค้าจาก API (ส่ง query params) + [NEW] มี Mock data รองรับกรณี API พัง
async function fetchProducts() {
  loading.value = true
  const q = new URLSearchParams()
  if (search.value)     q.set('search',   search.value)
  if (catFilter.value) q.set('category', catFilter.value)
  
  try {
    const res = await fetch(`/api/products?${q}`)
    if (!res.ok) throw new Error('API Error')
    products.value = await res.json()
  } catch (err) {
    // [NEW] Fallback Mock Data เผื่อเอาไว้ทดสอบตอนไม่มีหลังบ้านจริง
    console.warn("กำลังใช้ข้อมูลจำลองเนื่องจากติดต่อ API ไม่ได้:", err.message)
    if (products.value.length === 0) {
      products.value = [
        { id: 1, name: 'MacBook Air M3', category: 'Electronics', price: 34900, stock: 15, description: 'Apple M3 chip, 8GB Unified Memory, 256GB SSD' },
        { id: 2, name: 'Mechanical Keyboard G Pro', category: 'Electronics', price: 4290, stock: 3, description: 'RGB Backlit, GX Blue Clicky Switches' },
        { id: 3, name: 'Running Shoes Ultra', category: 'Footwear', price: 5500, stock: 0, description: 'Lightweight cushioned sole for marathon' },
        { id: 4, name: 'Coffee Beans Arabica', category: 'Food', price: 350, stock: 45, description: 'Medium roast 100% organic beans' }
      ]
    }
  } finally {
    loading.value = false
  }
}

// [NEW] ฟังก์ชันปรับสต็อกด่วน (+/- จากหน้าการ์ดสินค้า)
async function quickAdjustStock(product, amount) {
  const newStock = Math.max(0, product.stock + amount)
  // อัปเดตใน Client ทันทีเพื่อความลื่นไหลของ UI
  product.stock = newStock
  
  // ส่งไปบันทึกที่หลังบ้าน
  try {
    await fetch(`/api/products/${product.id}`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ ...product, price: parseFloat(product.price), stock: parseInt(newStock) })
    })
  } catch (err) {
    console.error("ไม่สามารถบันทึกการปรับสต็อกด่วนได้:", err)
  }
}

// เปิด modal แบบ Add
function openAdd() {
  editingId.value = null
  form.value = { name: '', category: '', price: '', stock: 0, description: '' }
  showModal.value = true
}

// เปิด modal แบบ Edit
function openEdit(p) {
  editingId.value = p.id
  form.value = {
    name: p.name, category: p.category,
    price: p.price, stock: p.stock,
    description: p.description || ''
  }
  showModal.value = true
}

// บันทึกข้อมูลสินค้า
async function saveProduct() {
  const body = {
    name:        form.value.name,
    category:    form.value.category,
    price:       parseFloat(form.value.price),
    stock:       parseInt(form.value.stock),
    description: form.value.description
  }
  const url    = editingId.value ? `/api/products/${editingId.value}` : '/api/products'
  const method = editingId.value ? 'PUT' : 'POST'
  
  try {
    await fetch(url, { method, headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(body) })
  } catch (err) {
    // ถ้าใช้ mock data ให้ทำการอัปเดตใน array ตรงๆ
    if (method === 'POST') {
      products.value.push({ id: Date.now(), ...body })
    } else {
      const idx = products.value.findIndex(p => p.id === editingId.value)
      if (idx !== -1) products.value[idx] = { id: editingId.value, ...body }
    }
  }
  
  showModal.value = false
  fetchProducts()
}

// ลบสินค้า
async function deleteProduct(id) {
  try {
    await fetch(`/api/products/${id}`, { method: 'DELETE' })
  } catch {
    products.value = products.value.filter(p => p.id !== id)
  }
  confirmDelete.value = null
  fetchProducts()
}

// คืน CSS class ตาม stock level
function stockClass(s) {
  if (s <= 0) return 'out'    // หมด
  if (s < 10) return 'low'    // ใกล้หมด
  if (s < 30) return 'mid'    // ปานกลาง
  return 'high'               // เพียงพอ
}

// คืน CSS class ตาม category
function catClass(cat) {
  const m = {
    Electronics: 'c-elec', Clothing: 'c-cloth',
    Footwear:    'c-foot',  Food:     'c-food',
    Sports:      'c-sport', Books:    'c-book',
    Furniture:   'c-furn',  Bags:     'c-bag',
    Accessories: 'c-acc',   Tools:    'c-tool'
  }
  return m[cat] || 'c-other'
}

onMounted(fetchProducts)
</script>

<template>
  <div>

    <header class="app-header">
      <div class="logo">
        <span class="logo-icon">📦</span>
        <div>
          <div class="logo-name">StockPro</div>
          <div class="logo-sub">ระบบจัดการสินค้าคงคลัง</div>
        </div>
      </div>
      <button class="btn-add" @click="openAdd">+ เพิ่มสินค้า</button>
    </header>

    <main class="main">

      <div class="stats-grid">
        <div class="stat-card">
          <div class="stat-icon si-green">📦</div>
          <div class="stat-body">
            <div class="stat-val" style="color:#059669">{{ stats.total }}</div>
            <div class="stat-label">สินค้าทั้งหมด</div>
          </div>
        </div>
        <div class="stat-card">
          <div class="stat-icon si-red">⚠️</div>
          <div class="stat-body">
            <div class="stat-val" style="color:#dc2626">{{ stats.lowStock }}</div>
            <div class="stat-label">สต็อกใกล้หมด (&lt;10)</div>
          </div>
        </div>
        <div class="stat-card">
          <div class="stat-icon si-amber">📊</div>
          <div class="stat-body">
            <div class="stat-val" style="color:#d97706">{{ stats.totalItems.toLocaleString() }}</div>
            <div class="stat-label">จำนวนสต็อกรวม</div>
          </div>
        </div>
        <div class="stat-card">
          <div class="stat-icon si-blue">💰</div>
          <div class="stat-body">
            <div class="stat-val" style="color:#2563eb;font-size:1.3rem">
              ฿{{ Math.round(stats.totalValue).toLocaleString() }}
            </div>
            <div class="stat-label">มูลค่าสต็อกรวม</div>
          </div>
        </div>
        <div class="stat-card">
          <div class="stat-icon si-purple">🏷️</div>
          <div class="stat-body">
            <div class="stat-val" style="color:#7c3aed">
              ฿{{ Math.round(stats.avgPrice).toLocaleString() }}
            </div>
            <div class="stat-label">ราคาเฉลี่ยต่อรายการ</div>
          </div>
        </div>
      </div>

      <div class="alert-low" v-if="stats.lowStock > 0">
        🚨 มีสินค้า <strong>{{ stats.lowStock }} รายการ</strong>
        ที่มีจำนวนสต็อกน้อยกว่า 10 ชิ้น — กรุณาตรวจสอบและเติมสต็อก
      </div>

      <div class="toolbar">
        <input
          v-model="search"
          @input="debounceFetch"
          class="input-search"
          placeholder="🔍 ค้นหาชื่อสินค้า หรือคำอธิบาย..."
        />
        <select v-model="catFilter" @change="fetchProducts" class="input-select">
          <option value="">ทุกหมวดหมู่</option>
          <option v-for="c in categories" :key="c" :value="c">{{ c }}</option>
        </select>
        
        <select v-model="stockFilter" class="input-select">
          <option value="">สถานะสต็อกทั้งหมด</option>
          <option value="high">🟢 มีเพียงพอ (&gt;= 30)</option>
          <option value="mid">🟡 ปานกลาง (10-29)</option>
          <option value="low">🟠 ใกล้หมด (&lt; 10)</option>
          <option value="out">🔴 หมดแล้ว (0)</option>
        </select>

        <select v-model="sortBy" class="input-select">
          <option value="name-asc">🔤 ชื่อสินค้า (A - Z)</option>
          <option value="name-desc">🔤 ชื่อสินค้า (Z - A)</option>
          <option value="price-asc">💵 ราคา: ต่ำ ➔ สูง</option>
          <option value="price-desc">💵 ราคา: สูง ➔ ต่ำ</option>
          <option value="stock-asc">📦 สต็อก: น้อย ➔ มาก</option>
          <option value="stock-desc">📦 สต็อก: มาก ➔ น้อย</option>
        </select>

        <span class="result-count" v-if="!loading">
          แสดง {{ filtered.length }} / {{ products.length }} รายการ
        </span>
      </div>

      <div class="state-box" v-if="loading">
        <div class="state-icon">⏳</div>
        <div>กำลังโหลดข้อมูลสินค้า...</div>
      </div>

      <div class="state-box" v-else-if="filtered.length === 0">
        <div class="state-icon">📭</div>
        <div class="state-title">ไม่พบสินค้า</div>
        <div class="state-desc">
          {{ search || catFilter || stockFilter ? 'ลองเปลี่ยน keyword หรือ filter' : 'กดปุ่ม "+ เพิ่มสินค้า" เพื่อเริ่มต้น' }}
        </div>
      </div>

      <div class="product-grid" v-else>
        <div
          v-for="p in filtered"
          :key="p.id"
          class="product-card"
          :class="{ 'card-low': p.stock > 0 && p.stock < 10, 'card-out': p.stock <= 0 }"
        >
          <div class="card-top">
            <span class="cat-badge" :class="catClass(p.category)">{{ p.category }}</span>
            <div class="product-name">{{ p.name }}</div>
            <div class="product-desc" v-if="p.description">{{ p.description }}</div>
            <div class="product-price">
              ฿{{ parseFloat(p.price).toLocaleString('th-TH', { minimumFractionDigits: 2 }) }}
            </div>
            
            <div class="stock-info">
              <div class="stock-row">
                <span class="stock-label">สต็อก</span>
                <span class="stock-num" :class="stockClass(p.stock)">
                  {{ p.stock.toLocaleString() }} ชิ้น
                  <span v-if="p.stock <= 0"> — หมดแล้ว!</span>
                  <span v-else-if="p.stock < 10"> — ใกล้หมด!</span>
                </span>
              </div>
              <div class="stock-track">
                <div
                  class="stock-fill"
                  :class="stockClass(p.stock)"
                  :style="{ width: Math.min((p.stock / 80) * 100, 100) + '%' }"
                ></div>
              </div>
            </div>

            <div class="quick-stock-adjust">
              <button class="btn-quick-minus" @click.stop="quickAdjustStock(p, -1)" :disabled="p.stock <= 0">− ลด 1</button>
              <button class="btn-quick-plus" @click.stop="quickAdjustStock(p, 1)">+ เพิ่ม 1</button>
            </div>

          </div>
          <div class="card-footer">
            <button class="btn-edit" @click="openEdit(p)">✏️ แก้ไข</button>
            <button class="btn-del" @click="confirmDelete = p">🗑️ ลบ</button>
          </div>
        </div>
      </div>

    </main>

    <div class="overlay" v-if="showModal" @click.self="showModal = false">
      <div class="modal">
        <div class="modal-title">
          {{ editingId ? '✏️ แก้ไขสินค้า' : '📦 เพิ่มสินค้าใหม่' }}
        </div>
        <form @submit.prevent="saveProduct">
          <div class="form-group">
            <label class="form-label">ชื่อสินค้า *</label>
            <input class="form-input" v-model="form.name" placeholder="เช่น MacBook Air M3" required />
          </div>
          <div class="form-row">
            <div class="form-group">
              <label class="form-label">หมวดหมู่ *</label>
              <input class="form-input" v-model="form.category" list="cat-list" placeholder="เช่น Electronics" required />
              <datalist id="cat-list">
                <option v-for="c in categories" :value="c" :key="c" />
              </datalist>
            </div>
            <div class="form-group">
              <label class="form-label">ราคา (บาท) *</label>
              <input class="form-input" v-model="form.price" type="number" min="0" step="0.01" placeholder="0.00" required />
            </div>
          </div>
          <div class="form-group">
            <label class="form-label">จำนวนสต็อก (ชิ้น) *</label>
            <input class="form-input" v-model="form.stock" type="number" min="0" placeholder="0" required />
          </div>
          <div class="form-group">
            <label class="form-label">คำอธิบายสินค้า</label>
            <textarea class="form-input" v-model="form.description" rows="3" placeholder="รายละเอียดเพิ่มเติม (ถ้ามี)" style="resize:vertical"></textarea>
          </div>
          <div class="modal-footer">
            <button type="button" class="btn-cancel" @click="showModal = false">ยกเลิก</button>
            <button type="submit" class="btn-save">
              {{ editingId ? 'บันทึกการเปลี่ยนแปลง' : 'เพิ่มสินค้า' }}
            </button>
          </div>
        </form>
      </div>
    </div>

    <div class="overlay" v-if="confirmDelete" @click.self="confirmDelete = null">
      <div class="modal confirm">
        <div class="confirm-icon">🗑️</div>
        <div class="confirm-title">ยืนยันการลบสินค้า</div>
        <div class="confirm-desc">
          คุณต้องการลบ <strong>{{ confirmDelete?.name }}</strong> ออกจากระบบหรือไม่?<br>
          <span style="color:#dc2626">การกระทำนี้ไม่สามารถย้อนกลับได้</span>
        </div>
        <div class="confirm-actions">
          <button class="btn-cancel" @click="confirmDelete = null">ยกเลิก</button>
          <button class="btn-danger-confirm" @click="deleteProduct(confirmDelete.id)">ลบเลย</button>
        </div>
      </div>
    </div>

  </div>
</template>

<style scoped>
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

.app-header {
  position: sticky; top: 0; z-index: 100;
  background: #fff; border-bottom: 1px solid #e2e8f0;
  height: 62px; padding: 0 1.5rem;
  display: flex; align-items: center; gap: .85rem;
  box-shadow: 0 1px 6px rgba(0,0,0,.07);
}
.logo { display: flex; align-items: center; gap: .6rem; }
.logo-icon { font-size: 1.6rem; }
.logo-name { font-weight: 800; font-size: 1.15rem; color: #065f46; line-height: 1; }
.logo-sub  { font-size: .72rem; color: #64748b; }
.btn-add {
  margin-left: auto;
  background: #10b981; color: #fff;
  border: none; border-radius: 8px;
  padding: .55rem 1.2rem; font-size: .9rem; font-weight: 700;
  cursor: pointer; transition: background .2s;
}
.btn-add:hover { background: #059669; }

.main { max-width: 1280px; margin: 0 auto; padding: 1.75rem 1.5rem; }

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: 1rem; margin-bottom: 1.5rem;
}
.stat-card {
  background: #fff; border: 1px solid #e2e8f0; border-radius: 12px;
  padding: 1.1rem; display: flex; align-items: center; gap: .9rem;
}
.stat-icon {
  width: 46px; height: 46px; border-radius: 10px;
  display: flex; align-items: center; justify-content: center;
  font-size: 1.3rem; flex-shrink: 0;
}
.si-green  { background: #d1fae5; }
.si-red    { background: #fee2e2; }
.si-amber  { background: #fef3c7; }
.si-blue   { background: #dbeafe; }
.si-purple { background: #f3e8ff; } /* [NEW] สีสถิติแบบใหม่ */
.stat-val { font-size: 1.5rem; font-weight: 800; line-height: 1; }
.stat-label { font-size: .8rem; color: #64748b; margin-top: .2rem; }

.alert-low {
  background: #fef2f2; border: 1px solid #fca5a5;
  border-radius: 10px; padding: .85rem 1.2rem;
  font-size: .92rem; margin-bottom: 1.5rem; color: #991b1b;
}

.toolbar { display: flex; gap: .75rem; margin-bottom: 1.5rem; flex-wrap: wrap; align-items: center; }
.input-search {
  flex: 2; min-width: 200px;
  padding: .6rem 1rem; border: 1.5px solid #e2e8f0; border-radius: 8px;
  font-size: .95rem; outline: none; transition: border-color .2s;
}
.input-search:focus { border-color: #10b981; }
.input-select {
  flex: 1; min-width: 150px;
  padding: .6rem .9rem; border: 1.5px solid #e2e8f0; border-radius: 8px;
  background: #fff; font-size: .9rem; outline: none; cursor: pointer;
}
.input-select:focus { border-color: #10b981; }
.result-count { font-size: .82rem; color: #64748b; white-space: nowrap; margin-left: auto; }

.state-box { text-align: center; padding: 4rem 1rem; color: #64748b; }
.state-icon { font-size: 3rem; margin-bottom: .75rem; }
.state-title { font-size: 1.1rem; font-weight: 700; color: #1e293b; margin-bottom: .3rem; }
.state-desc { font-size: .9rem; }

.product-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(275px, 1fr));
  gap: 1.2rem;
}
.product-card {
  background: #fff; border: 1px solid #e2e8f0; border-radius: 14px;
  overflow: hidden; display: flex; flex-direction: column; transition: transform .2s, box-shadow .2s;
}
.product-card:hover { transform: translateY(-3px); box-shadow: 0 8px 24px rgba(0,0,0,.1); }
.product-card.card-low  { border-color: #fca5a5; }
.product-card.card-out  { border-color: #d1d5db; opacity: .75; }

.card-top { padding: 1.1rem; flex: 1; }
.cat-badge {
  display: inline-block; padding: .2rem .65rem;
  border-radius: 20px; font-size: .72rem; font-weight: 700; margin-bottom: .6rem;
}
.c-elec  { background: #dbeafe; color: #1e40af; }
.c-cloth { background: #fce7f3; color: #9d174d; }
.c-foot  { background: #f3e8ff; color: #6b21a8; }
.c-food  { background: #fef3c7; color: #92400e; }
.c-sport { background: #d1fae5; color: #065f46; }
.c-book  { background: #e0f2fe; color: #0369a1; }
.c-furn  { background: #fdf4ff; color: #7e22ce; }
.c-bag   { background: #fff7ed; color: #9a3412; }
.c-acc   { background: #f0fdf4; color: #15803d; }
.c-tool  { background: #f1f5f9; color: #475569; }
.c-other { background: #ede9fe; color: #5b21b6; }

.product-name  { font-size: 1rem; font-weight: 700; color: #1e293b; margin-bottom: .3rem; line-height: 1.35; }
.product-desc  { font-size: .82rem; color: #64748b; line-height: 1.5; margin-bottom: .75rem; }
.product-price { font-size: 1.25rem; font-weight: 800; color: #059669; }

.stock-info { margin-top: .85rem; }
.stock-row  { display: flex; justify-content: space-between; font-size: .82rem; margin-bottom: .3rem; }
.stock-label { color: #64748b; }
.stock-num   { font-weight: 700; }
.stock-num.out  { color: #9ca3af; }
.stock-num.low  { color: #dc2626; }
.stock-num.mid  { color: #d97706; }
.stock-num.high { color: #059669; }
.stock-track { height: 6px; background: #f1f5f9; border-radius: 3px; overflow: hidden; }
.stock-fill  { height: 100%; border-radius: 3px; transition: width .4s ease; min-width: 4px; }
.stock-fill.out  { background: #d1d5db; width: 2% !important; }
.stock-fill.low  { background: #dc2626; }
.stock-fill.mid  { background: #f59e0b; }
.stock-fill.high { background: #10b981; }

/* [NEW] Quick Stock Adjust CSS */
.quick-stock-adjust {
  display: flex; gap: .4rem; margin-top: 1rem;
}
.btn-quick-minus, .btn-quick-plus {
  flex: 1; padding: .35rem; border-radius: 6px; border: 1px solid #e2e8f0;
  font-size: .78rem; font-weight: 600; cursor: pointer; background: #fff; transition: all .15s;
}
.btn-quick-minus:hover:not(:disabled) { background: #fee2e2; color: #dc2626; border-color: #fca5a5; }
.btn-quick-plus:hover { background: #e0f2fe; color: #0369a1; border-color: #bae6fd; }
.btn-quick-minus:disabled { opacity: 0.4; cursor: not-allowed; }

.card-footer {
  display: flex; gap: .5rem;
  padding: .75rem 1.1rem;
  border-top: 1px solid #f1f5f9; background: #fafafa;
}
.btn-edit, .btn-del {
  flex: 1; padding: .45rem; border-radius: 7px;
  font-size: .82rem; font-weight: 600; cursor: pointer; border: none; transition: all .2s;
}
.btn-edit { background: #f1f5f9; color: #334155; }
.btn-edit:hover { background: #e2e8f0; }
.btn-del  { background: #fee2e2; color: #dc2626; }
.btn-del:hover  { background: #fecaca; }

.overlay {
  position: fixed; inset: 0;
  background: rgba(0,0,0,.45);
  display: flex; align-items: center; justify-content: center;
  z-index: 500; padding: 1rem;
}
.modal {
  background: #fff; border-radius: 16px;
  width: 100%; max-width: 480px;
  max-height: 90vh; overflow-y: auto; padding: 2rem;
}
.modal-title { font-size: 1.2rem; font-weight: 700; margin-bottom: 1.5rem; color: #1e293b; }

.form-group { margin-bottom: 1rem; }
.form-label { display: block; font-size: .87rem; font-weight: 600; margin-bottom: .35rem; }
.form-input {
  width: 100%; padding: .6rem .85rem;
  border: 1.5px solid #e2e8f0; border-radius: 8px;
  font-size: .95rem; outline: none; transition: border-color .2s;
}
.form-input:focus { border-color: #10b981; }
.form-row { display: grid; grid-template-columns: 1fr 1fr; gap: .75rem; }

.modal-footer {
  display: flex; gap: .75rem; justify-content: flex-end;
  padding-top: 1rem; border-top: 1px solid #f1f5f9; margin-top: 1rem;
}
.btn-cancel {
  background: #f1f5f9; color: #334155;
  border: none; border-radius: 8px; padding: .6rem 1.25rem;
  font-weight: 600; cursor: pointer;
}
.btn-cancel:hover { background: #e2e8f0; }
.btn-save {
  background: #10b981; color: #fff;
  border: none; border-radius: 8px; padding: .6rem 1.25rem;
  font-weight: 700; cursor: pointer;
}
.btn-save:hover { background: #059669; }

.confirm { text-align: center; max-width: 380px; }
.confirm-icon  { font-size: 3rem; margin-bottom: 1rem; }
.confirm-title { font-size: 1.1rem; font-weight: 700; margin-bottom: .5rem; }
.confirm-desc  { font-size: .9rem; color: #64748b; margin-bottom: 1.5rem; line-height: 1.6; }
.confirm-actions { display: flex; gap: .75rem; justify-content: center; }
.btn-danger-confirm {
  background: #dc2626; color: #fff;
  border: none; border-radius: 8px; padding: .6rem 1.4rem;
  font-weight: 700; cursor: pointer;
}
.btn-danger-confirm:hover { background: #b91c1c; }

@media (max-width: 768px) {
  .toolbar { flex-direction: column; align-items: stretch; }
  .result-count { margin-left: 0; text-align: center; }
}
@media (max-width: 640px) {
  .form-row { grid-template-columns: 1fr; }
  .stats-grid { grid-template-columns: repeat(auto-fit, minmax(140px, 1fr)); }
}
</style>

```