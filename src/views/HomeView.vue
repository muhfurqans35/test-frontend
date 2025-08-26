<template>
  <v-container class="py-6">
    <!-- Snackbar -->
    <v-snackbar v-model="snackbar.show" :color="snackbar.color" location="top" class="rounded-md">
      {{ snackbar.message }}
      <template #actions>
        <v-btn variant="text" @click="snackbar.show = false">Tutup</v-btn>
      </template>
    </v-snackbar>

    <!-- Header -->
    <v-row align="center" justify="space-between">
      <v-col cols="6">
        <h1 class="text-h5 font-extrabold">Daftar Diskon</h1>
        <div v-if="discounts.length" class="text-subtitle-1 font-medium text-gray-500">
          Total jumlah diskon: {{ totalDiscountCount }}
        </div>
      </v-col>
      <v-col cols="6" class="text-end">
        <v-btn color="success" variant="flat" rounded="pill" @click="openModal">
          <Plus class="w-4 h-4 mr-1" /> Tambah Diskon
        </v-btn>
        <v-btn v-if="selected.length" color="error" variant="flat" rounded="pill" class="ms-2 normal-case"
          @click="openDeleteMultipleModal">
          Hapus
        </v-btn>
      </v-col>
    </v-row>

    <!-- Filter -->
    <v-row align="center" dense>
      <v-col :cols="discounts.length ? 4 : 0" v-if="discounts.length">
        <v-text-field v-model="search" placeholder="Cari Diskon..." variant="outlined" rounded="pill" density="compact"
          clearable>
          <template #prepend-inner>
            <Search class="w-4 h-4 text-gray-500" />
          </template>
        </v-text-field>
      </v-col>

      <v-col :cols="discounts.length ? 3 : 4">
        <v-select v-model="selectedStore" :items="stores" item-title="nama_toko" item-value="_id" placeholder="Pilih Toko"
          variant="outlined" rounded="lg" density="compact" clearable>
          <template #prepend-inner>
            <Store class="w-4 h-4 text-gray-500" />
          </template>
        </v-select>
      </v-col>
    </v-row>

    <!-- Table -->
    <v-data-table v-model="selected" :items="filteredDiscounts" return-object :headers="headers" item-key="_id"
      class="elevation-0 green-checkbox" :items-per-page="10" :hide-default-footer="!filteredDiscounts.length"
      :show-select="filteredDiscounts.length > 0" :hide-default-header="!filteredDiscounts.length" rounded="md" border z>
      <template #item.nama_diskon="{ item }">
        <div class="flex items-center gap-2">
          <span>{{ item.nama_diskon }}</span>
          <v-chip v-if="item._id === newItemId" size="x-small" variant="flat"
            class="rounded-md bg-white text-blue border">
            baru
          </v-chip>
        </div>
      </template>
      <template #item.jumlah="{ item }">
        <span v-if="item.type === 'percentage'">{{ item.jumlah }}%</span>
        <span v-else>Rp {{ item.jumlah.toLocaleString('id-ID') }}</span>
      </template>
      <template #item.actions="{ item }">
        <v-btn variant="text" @click="editItem(item)">
          <PencilLine size=18 />
        </v-btn>
      </template>
      <template #no-data>
        <v-card class="py-15 text-center elevation-0 border rounded-md">
          <img src="@/assets/Layer_1.svg" class="mx-auto mb-2" />
          <div class="text-subtitle-1 font-medium">Belum ada diskon</div>
          <div class="text-subtitle-2 font-medium">
            Silahkan tambahkan diskon untuk menarik pelanggan <br />dan meningkatkan penjualan
          </div>
        </v-card>
      </template>
    </v-data-table>

    <!-- Modal Tambah/Edit Diskon -->
    <v-dialog v-model="modal" max-width="500">
      <v-card class="rounded-xl pa-4">
        <v-card-title class="d-flex justify-space-between align-center pa-0 mb-4">
          <span class="text-lg font-bold">{{ editMode ? 'Ubah Diskon' : 'Tambah Diskon' }}</span>
          <v-btn icon variant="text" size="small" @click="closeModal">
            <X size="18" />
          </v-btn>
        </v-card-title>

        <v-form ref="formRef" v-model="formValid">
          <v-card-text class="pa-0">
            <div class="mb-4">
              <v-text-field v-model="form.nama_diskon" label="Nama Diskon"
                placeholder="Misal: Diskon opening, diskon akhir tahun" variant="outlined" rounded="lg"
                :rules="[validateNama]" :error-messages="nameError" />
            </div>
            <div class="mb-4">
              <div class="d-flex gap-4">
                <v-text-field v-model.number="form.jumlah" label="Diskon" placeholder="0" type="number" variant="outlined"
                  rounded="lg" :rules="[validateJumlah]" min="0" class="flex-1"
                  :max="form.type === 'percentage' ? 100 : undefined">
                  <template v-if="form.type === 'percentage'" #append-inner>
                    <span class="text-gray-500 font-medium">%</span>
                  </template>
                  <template v-else #prepend-inner>
                    <span class="text-gray-500 font-medium">Rp</span>
                  </template>
                </v-text-field>
                <v-btn-toggle v-model="form.type" mandatory rounded="pill">
                  <v-btn value="percentage" variant="outlined" class="text-none"
                    :class="form.type === 'percentage' ? 'bg-white border border-success text-success' : ''">
                    <Check v-if="form.type === 'percentage'" class="mr-1" size="14" /> %
                  </v-btn>
                  <v-btn value="nominal" variant="outlined" class="text-none"
                    :class="form.type === 'nominal' ? 'bg-white border border-success text-success' : ''">
                    <Check v-if="form.type === 'nominal'" class="mr-1" size="14" /> Rp
                  </v-btn>
                </v-btn-toggle>
              </div>
            </div>
          </v-card-text>

          <v-card-actions class="pa-0 d-flex justify-space-between align-center">
            <v-btn v-if="editMode" color="error" variant="outlined" class="rounded-pill px-5"
              @click="openDeleteModal(form)">
              Hapus
            </v-btn>
            <v-btn color="success" variant="flat" class="rounded-pill text-white" :class="editMode ? 'px-6' : 'w-100'"
              @click="saveItem" :disabled="!formValid || !!nameError || !selectedStore">
              Simpan
            </v-btn>
          </v-card-actions>
        </v-form>
      </v-card>
    </v-dialog>

    <!-- Modal Konfirmasi Hapus Satu Item -->
    <v-dialog v-model="deleteModal" max-width="600">
      <v-card class="rounded-xl pa-4">
        <v-card-title><span class="text-lg font-bold pa-0 mb-4">Hapus Diskon</span></v-card-title>
        <v-card-text class="text-subtitle-1 mb-4">
          Apakah Anda yakin ingin menghapus diskon "{{ deleteTargetName }}"?
          <ul class="ml-6 list-disc">
            <li>Diskon yang dihapus tidak bisa dikembalikan lagi.</li>
          </ul>
        </v-card-text>
        <v-card-actions class="d-flex justify-end gap-2 pa-0">
          <v-btn color="gray" variant="outlined" class="rounded-pill text-red px-6" @click="closeDeleteModal">
            Batalkan
          </v-btn>
          <v-btn color="error" variant="flat" class="rounded-pill text-white px-6" @click="deleteItemConfirmed">
            Hapus
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <!-- Modal Konfirmasi Hapus Banyak Item -->
    <v-dialog v-model="deleteMultipleModal" max-width="600">
      <v-card class="rounded-xl pa-4">
        <v-card-title><span class="text-lg font-bold pa-0 mb-4">Hapus Diskon</span></v-card-title>
        <v-card-text class="text-subtitle-1 mb-4">
          Apakah Anda yakin ingin menghapus diskon yang dipilih?
          <ul class="ml-6 list-disc">
            <li>Diskon yang dihapus tidak bisa dikembalikan lagi.</li>
          </ul>
        </v-card-text>
        <v-card-actions class="d-flex justify-end gap-2 pa-0">
          <v-btn color="gray" variant="outlined" class="rounded-pill text-red px-6" @click="closeDeleteMultipleModal">
            Batalkan
          </v-btn>
          <v-btn color="error" variant="flat" class="rounded-pill text-white px-6" @click="deleteSelectedConfirmed">
            Hapus
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

  </v-container>
</template>

<script setup>
import { ref, computed, onMounted, watch } from 'vue'
import axios from 'axios'
import { Plus, PencilLine, Search, Store, X, Check } from 'lucide-vue-next'

const API_BASE_URL = 'https://crudcrud.com/api/be9148f736c241b4810f4e09f01dcf90'
const DISCOUNT_API_URL = `${API_BASE_URL}/diskon`
const STORE_API_URL = `${API_BASE_URL}/toko`

const newItemId = ref(null)
const discounts = ref([])
const stores = ref([])
const search = ref('')
const selectedStore = ref('')
const selected = ref([])
const modal = ref(false)
const editMode = ref(false)
const form = ref({ _id: null, nama_diskon: '', jumlah: null, toko_id: '', type: 'percentage' })
const formValid = ref(false)
const formRef = ref(null)
const snackbar = ref({ show: false, message: '', color: 'success' })
const nameError = ref('')

// Hapus item
const deleteModal = ref(false)
const deleteTarget = ref(null)
const deleteTargetName = ref('')

// Hapus banyak
const deleteMultipleModal = ref(false)

const headers = [
  { title: 'Nama Diskon', key: 'nama_diskon', align: 'start' },
  { title: 'Nilai Diskon', key: 'jumlah', align: 'start' },
  { title: '', key: 'actions', align: 'start', sortable: false },
]

const filteredDiscounts = computed(() =>
  discounts.value.filter(
    d => (!selectedStore.value || d.toko_id === selectedStore.value) &&
      d.nama_diskon.toLowerCase().includes(search.value.toLowerCase())
  )
)
const totalDiscountCount = computed(() => discounts.value.length)

// Validasi
const validateNama = (value) => {
  if (!value || value.trim() === '') return 'Nama diskon wajib diisi'
  if (value.trim().length < 3) return 'Nama diskon minimal 3 karakter'
  return true
}
const validateJumlah = (value) => {
  if (value === null || value === '' || value === undefined) return 'Nilai diskon wajib diisi'
  if (value <= 0) return 'Nilai diskon harus lebih dari 0'
  if (form.value.type === 'percentage' && value > 100) return 'Persentase tidak boleh lebih dari 100%'
  return true
}

watch(() => form.value.nama_diskon, (newName) => {
  if (newName && newName.trim()) {
    nameError.value = discounts.value.some(d => d.nama_diskon.toLowerCase() === newName.toLowerCase() && d._id !== form.value._id) ? 'Nama diskon sudah ada' : ''
  } else {
    nameError.value = ''
  }
})

async function fetchStores() {
  try { const res = await axios.get(STORE_API_URL); stores.value = res.data }
  catch { snackbar.value = { show: true, message: 'Gagal mengambil data toko!', color: 'error' } }
}
async function fetchDiscounts() {
  try { const res = await axios.get(DISCOUNT_API_URL); discounts.value = res.data }
  catch { snackbar.value = { show: true, message: 'Gagal mengambil data diskon!', color: 'error' } }
}

function openModal() {
  if (!selectedStore.value) {
    snackbar.value = { show: true, message: 'Pilih toko terlebih dahulu di filter!', color: 'warning' }
    return
  }
  editMode.value = false
  form.value = { _id: null, nama_diskon: '', jumlah: null, toko_id: selectedStore.value, type: 'percentage' }
  nameError.value = ''
  modal.value = true
}

function closeModal() { modal.value = false; nameError.value = ''; formRef.value?.resetValidation() }

function editItem(item) { editMode.value = true; form.value = { ...item }; nameError.value = ''; modal.value = true }

async function saveItem() {
  const { valid } = await formRef.value.validate()
  if (!valid || nameError.value) { snackbar.value = { show: true, message: 'Mohon perbaiki kesalahan pada form!', color: 'error' }; return }
  try {
    const payload = { nama_diskon: form.value.nama_diskon.trim(), jumlah: form.value.jumlah, toko_id: form.value.toko_id, type: form.value.type }
    if (editMode.value) {
      await axios.put(`${DISCOUNT_API_URL}/${form.value._id}`, payload)
      const idx = discounts.value.findIndex(d => d._id === form.value._id)
      if (idx !== -1) discounts.value[idx] = { ...form.value, ...payload }
      snackbar.value = { show: true, message: `Diskon "${payload.nama_diskon}" berhasil diperbarui!`, color: 'success' }
    } else {
      const res = await axios.post(DISCOUNT_API_URL, payload)
      discounts.value.push(res.data)
      newItemId.value = res.data._id
      snackbar.value = { show: true, message: `Diskon "${payload.nama_diskon}" berhasil ditambahkan!`, color: 'success' }
    }
    closeModal()
  } catch { snackbar.value = { show: true, message: 'Gagal menyimpan diskon!', color: 'error' } }
}

// Hapus satu
function openDeleteModal(item) { deleteTarget.value = item; deleteTargetName.value = item.nama_diskon; deleteModal.value = true }
function closeDeleteModal() { deleteModal.value = false; deleteTarget.value = null; deleteTargetName.value = '' }
async function deleteItemConfirmed() {
  if (!deleteTarget.value) return
  try { await axios.delete(`${DISCOUNT_API_URL}/${deleteTarget.value._id}`); discounts.value = discounts.value.filter(d => d._id !== deleteTarget.value._id); snackbar.value = { show: true, message: `Diskon "${deleteTarget.value.nama_diskon}" berhasil dihapus!`, color: 'success' } }
  catch { snackbar.value = { show: true, message: 'Gagal menghapus diskon!', color: 'error' } }
  finally { closeDeleteModal() }
}

// Hapus banyak
function openDeleteMultipleModal() { deleteMultipleModal.value = true }
function closeDeleteMultipleModal() { deleteMultipleModal.value = false }
async function deleteSelectedConfirmed() {
  try {
    for (const item of selected.value) { await axios.delete(`${DISCOUNT_API_URL}/${item._id}`) }
    const deletedIds = selected.value.map(i => i._id)
    discounts.value = discounts.value.filter(d => !deletedIds.includes(d._id))
    snackbar.value = { show: true, message: `${selected.value.length} diskon berhasil dihapus!`, color: 'success' }
    selected.value = []
  } catch { snackbar.value = { show: true, message: 'Gagal menghapus diskon!', color: 'error' } }
  finally { closeDeleteMultipleModal() }
}

onMounted(async () => { await fetchStores(); await fetchDiscounts() })
</script>
