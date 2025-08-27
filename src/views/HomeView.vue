<template>
  <div class="py-6 px-4 md:px-6">
    <!-- Toast -->
    <div class="fixed top-4 left-1/2 transform -translate-x-1/2 z-50 space-y-2">
      <div v-for="toast in toasts" :key="toast.id" :class="[
        'flex items-center gap-3 px-4 py-3 rounded-md shadow-lg transition-all duration-300',
        toast.type === 'success'
          ? 'bg-green-600 text-white'
          : toast.type === 'error'
            ? 'bg-red-600 text-white'
            : toast.type === 'warning'
              ? 'bg-orange-600 text-white'
              : 'bg-blue-600 text-white',
      ]">
        <component :is="getLucideIcon(toast.type)" class="w-5 h-5" />
        <span class="text-sm font-medium">{{ toast.message }}</span>
        <button @click="removeToast(toast.id)"
          class="ml-2 text-white hover:bg-black hover:bg-opacity-10 rounded p-1 transition-colors">
          <X class="w-4 h-4" />
        </button>
      </div>
    </div>

    <!-- Header -->
    <div class="grid grid-cols-2 items-center gap-4 mb-6">
      <div>
        <h1 class="text-xl md:text-2xl font-bold text-gray-900">
          Daftar Diskon
        </h1>
        <div v-if="discounts.length" class="text-base font-medium text-gray-500 mt-1">
          Total jumlah diskon: {{ totalDiscountCount }}
        </div>
      </div>
      <div class="text-right flex justify-end">
        <button @click="openModal"
          class="inline-flex items-center gap-2 px-4 py-2 bg-green-600 text-white rounded-full hover:bg-green-700 transition-colors font-medium text-sm">
          <Plus class="w-4 h-4" />
          Tambah Diskon
        </button>
        <button v-if="selected.length" @click="openDeleteMultipleModal"
          class="inline-flex items-center gap-1 px-4 py-2 bg-red-600 text-white rounded-full hover:bg-red-700 transition-colors font-medium text-sm ml-2">
          Hapus ({{ selected.length }})
        </button>
      </div>
    </div>

    <!-- Filter -->
    <div class="grid grid-cols-1 md:grid-cols-12 gap-4 mb-6 items-end">
      <div v-if="discounts.length" class="md:col-span-3">
        <md-outlined-text-field placeholder="Cari Diskon..." class="w-full" v-model="search" @input="handleSearch"
          style="--md-outlined-text-field-container-shape: 28px;">
          <Search slot="leading-icon" class="w-4 h-4 text-gray-500" />
        </md-outlined-text-field>
      </div>

      <div class="md:col-span-3">
        <md-outlined-select class="w-full" v-model="selectedStore" @change="handleStoreChange"
          style="--md-outlined-text-field-container-shape: 28px;">
          <div slot="leading-icon" class="flex items-center justify-center w-5 h-5">
            <Store class="w-4 h-4 text-gray-500" />
          </div>
          <md-select-option value="">
            <div slot="headline">Semua Toko</div>
          </md-select-option>
          <md-select-option v-for="store in stores" :key="store._id" :value="store._id">
            <div slot="headline">{{ store.nama_toko }}</div>
          </md-select-option>
        </md-outlined-select>
      </div>
    </div>

    <!-- Table -->
    <div class="bg-white rounded-lg border border-gray-200 shadow-sm overflow-hidden">
      <div class="overflow-x-auto">
        <table class="w-full" v-if="filteredDiscounts.length">
          <thead class="bg-gray-50 border-b border-gray-200">
            <tr>
              <th class="px-4 py-3 text-left w-12">
                <input type="checkbox" @change="toggleSelectAll" :indeterminate.prop="isIndeterminate"
                  :checked="isAllSelected" class="w-4 h-4 border-black rounded focus:ring-green-500 accent-green-600" />
              </th>
              <th class="px-4 py-3 text-left text-medium font-semibold text-gray-900">
                <button @click="handleSort('nama_diskon')"
                  class="flex items-center gap-1 hover:text-gray-700 transition-colors group">
                  <span>Nama Diskon</span>
                  <div class="flex flex-col text-xs">
                    <ChevronsUpDown size="12" v-if="sortField !== 'nama_diskon'" />
                    <ArrowUp size="12" v-else-if="sortField === 'nama_diskon' && sortOrder === 'asc'" />
                    <ArrowDown size="12" v-else-if="sortField === 'nama_diskon' && sortOrder === 'desc'" />
                  </div>
                </button>
              </th>
              <th class="px-4 py-3 text-left text-medium font-semibold text-gray-900">
                <button @click="handleSort('jumlah')"
                  class="flex items-center gap-1 hover:text-gray-700 transition-colors group">
                  <span>Nilai Diskon</span>
                  <div class="flex flex-col text-xs">
                    <ChevronsUpDown size="12" v-if="sortField !== 'jumlah'" />
                    <ArrowUp size="12" v-else-if="sortField === 'jumlah' && sortOrder === 'asc'" />
                    <ArrowDown size="12" v-else-if="sortField === 'jumlah' && sortOrder === 'desc'" />
                  </div>
                </button>
              </th>
            </tr>
          </thead>
          <tbody class="divide-y divide-gray-200">
            <tr v-for="item in paginatedDiscounts" :key="item._id" class="hover:bg-gray-50 transition-colors">
              <td class="px-4 py-3">
                <input type="checkbox" :value="item._id" @change="toggleSelect(item._id)"
                  :checked="selected.includes(item._id)"
                  class="w-4 h-4 border-black rounded focus:ring-green-500 accent-green-600" />
              </td>
              <td class="px-4 py-3">
                <div class="flex items-center gap-2">
                  <span class="text-gray-900">{{ item.nama_diskon }}</span>
                  <span v-if="item._id === newItemId"
                    class="inline-flex items-center px-2 text-xs bg-blue-50 text-blue-600 rounded-full border">
                    baru
                  </span>
                </div>
              </td>
              <td class="px-4 py-3 text-gray-900">
                <span v-if="item.type === 'percentage'">{{ item.jumlah }}%</span>
                <span v-else>Rp {{ item.jumlah.toLocaleString('id-ID') }}</span>
              </td>
              <td class="px-4 py-3">
                <button @click="editItem(item)" class="p-2 text-gray-600 hover:bg-gray-100 rounded-lg transition-colors">
                  <PencilLine class="w-4 h-4" />
                </button>
              </td>
            </tr>
          </tbody>
        </table>

        <div v-if="!filteredDiscounts.length" class="py-16 px-4 text-center">
          <img src="@/assets/Layer_1.svg" class="mx-auto mb-2" />
          <div class="text-lg font-medium text-gray-900 mb-2">
            Belum ada diskon
          </div>
          <div class="text-sm font-medium text-gray-500 max-w-md mx-auto">
            Silahkan tambahkan diskon untuk menarik pelanggan<br />
            dan meningkatkan penjualan
          </div>
        </div>
      </div>
    </div>

    <!-- Modal Tambah/Ubah -->
    <md-dialog ref="modalRef" class="max-w-md w-full mx-auto my-auto">
      <div slot="headline" class="flex justify-between items-center p-4 border-b border-gray-200">
        <span class="text-lg font-bold">{{
          editMode ? "Ubah Diskon" : "Tambah Diskon"
        }}</span>
        <button @click="closeModal" class="p-2 text-gray-500 hover:bg-gray-100 rounded-lg transition-colors">
          <X class="w-4 h-4" />
        </button>
      </div>

      <form slot="content" @submit.prevent="saveItem" class="space-y-4 p-4">
        <md-outlined-text-field v-model="form.nama_diskon" label="Nama Diskon"
          placeholder="Misal: Diskon opening, diskon akhir tahun" class="w-full" :error="!!nameError"
          :error-text="nameError" required />
        <div class="flex gap-4 items-center">
          <md-outlined-text-field v-model.number="form.jumlah" label="Diskon" placeholder="0" type="number" class="flex-1"
            :error="!!jumlahError" :error-text="jumlahError" :min="0" :max="form.type === 'percentage' ? 100 : undefined">
            <span v-if="form.type === 'nominal'" slot="leading-icon" class="text-gray-500 text-sm pl-2">
              Rp
            </span>
            <span v-if="form.type === 'percentage'" slot="trailing-icon" class="text-gray-500 text-sm pr-2">
              %
            </span>
          </md-outlined-text-field>

          <div class="flex rounded-full border border-gray-300 overflow-hidden bg-white">
            <button type="button" @click="form.type = 'percentage'" :class="[
              'px-4 py-2 text-sm font-medium flex items-center gap-1 border-r border-gray-300 transition-colors',
              form.type === 'percentage'
                ? 'bg-green-600 text-white'
                : 'bg-white text-gray-700 hover:bg-gray-50',
            ]">
              <Check v-if="form.type === 'percentage'" class="w-3 h-3" /> %
            </button>
            <button type="button" @click="form.type = 'nominal'" :class="[
              'px-4 py-2 text-sm font-medium flex items-center gap-1 transition-colors',
              form.type === 'nominal'
                ? 'bg-green-600 text-white'
                : 'bg-white text-gray-700 hover:bg-gray-50',
            ]">
              <Check v-if="form.type === 'nominal'" class="w-3 h-3" /> Rp
            </button>
          </div>
        </div>
      </form>

      <div slot="actions" class="flex justify-between items-center gap-3 p-4 border-t border-gray-200">
        <button v-if="editMode" @click="openDeleteModal(form)"
          class="px-5 py-2 border border-red-600 text-red-600 rounded-full hover:bg-red-50 transition-colors font-medium text-sm">
          Hapus
        </button>
        <div v-else class="flex-1"></div>

        <button @click="saveItem" :disabled="!formValid || !selectedStore"
          class="px-6 py-2 bg-green-600 text-white rounded-full hover:bg-green-700 transition-colors font-medium text-sm disabled:opacity-50 disabled:cursor-not-allowed">
          Simpan
        </button>
      </div>
    </md-dialog>
    <!-- Modal Hapus Single -->
    <md-dialog ref="deleteModalRef" class="max-w-md w-full mx-auto my-auto">
      <div slot="headline" class="p-4 font-bold text-lg">Hapus Diskon</div>
      <div slot="content" class="p-4 text-gray-700">
        Apakah Anda yakin ingin menghapus diskon
        <span class="font-semibold text-red-600">"{{ deleteTargetName }}"</span>?
        <ul class="ml-6 list-disc">
          <li>Diskon yang dihapus tidak bisa dikembalikan lagi.</li>
        </ul>
      </div>
      <div slot="actions" class="flex justify-end gap-2 p-4 border-t border-gray-200">
        <button @click="closeDeleteModal" class="px-4 py-2 rounded-full border text-red-600 hover:bg-gray-50">
          Batalkan
        </button>
        <button @click="confirmDelete" class="px-4 py-2 bg-red-600 text-white rounded-full hover:bg-red-700">
          Hapus
        </button>
      </div>
    </md-dialog>

    <!-- Modal Hapus Multiple -->
    <md-dialog ref="deleteMultipleModalRef" class="max-w-md w-full mx-auto my-auto">
      <div slot="headline" class="p-4 font-bold text-lg">Hapus Beberapa Diskon</div>
      <div slot="content" class="p-4 text-gray-700">
        Apakah Anda yakin ingin menghapus
        <span class="font-semibold text-red-600">{{ selected.length }}</span>
        diskon terpilih?
        <ul class="ml-6 list-disc">
          <li>Diskon yang dihapus tidak bisa dikembalikan lagi.</li>
        </ul>
      </div>
      <div slot="actions" class="flex justify-end gap-2 p-4 border-t border-gray-200">
        <button @click="closeDeleteMultipleModal" class="px-4 py-2 rounded-full border text-red-600 hover:bg-gray-50">
          Batalkan
        </button>
        <button @click="confirmDeleteMultiple" class="px-4 py-2 bg-red-600 text-white rounded-full hover:bg-red-700">
          Hapus
        </button>
      </div>
    </md-dialog>

  </div>
</template>

<script setup>
import { ref, computed, onMounted } from "vue";
import axios from "axios";
import {
  Plus,
  PencilLine,
  Search,
  Store,
  X,
  Check,
  CheckCircle,
  AlertTriangle,
  Info,
  XCircle,
  ArrowDown,
  ArrowUp,
  ChevronsUpDown
} from "lucide-vue-next";

import "@material/web/textfield/outlined-text-field.js";
import "@material/web/select/outlined-select.js";
import "@material/web/select/select-option.js";
import "@material/web/dialog/dialog.js";

const API_BASE_URL = "https://crudcrud.com/api/47d2aa72b4fc4300967d09b5ac6c4e9c";
const DISCOUNT_API_URL = `${API_BASE_URL}/diskon`;
const STORE_API_URL = `${API_BASE_URL}/toko`;

const discounts = ref([]);
const stores = ref([]);
const search = ref("");
const selectedStore = ref("");
const selected = ref([]);
const newItemId = ref(null);

// Sorting
const sortField = ref(null);
const sortOrder = ref('asc');

const editMode = ref(false);
const form = ref({
  _id: null,
  nama_diskon: "",
  jumlah: null,
  toko_id: "",
  type: "percentage",
});

const nameError = ref("");
const jumlahError = ref("");

const deleteModalRef = ref(null);
const deleteMultipleModalRef = ref(null);
const deleteTarget = ref(null);
const deleteTargetName = ref("");

function openDeleteModal(item) {
  deleteTarget.value = item;
  deleteTargetName.value = item.nama_diskon;
  deleteModalRef.value?.show();
}

function closeDeleteModal() {
  deleteModalRef.value?.close();
}

async function confirmDelete() {
  try {
    await axios.delete(`${DISCOUNT_API_URL}/${deleteTarget.value._id}`);
    discounts.value = discounts.value.filter(d => d._id !== deleteTarget.value._id);
    showToast(`Diskon "${deleteTargetName.value}" berhasil dihapus!`, "success");
  } catch {
    showToast("Gagal menghapus diskon!", "error");
  } finally {
    closeDeleteModal();
  }
}

function openDeleteMultipleModal() {
  deleteMultipleModalRef.value?.show();
}

function closeDeleteMultipleModal() {
  deleteMultipleModalRef.value?.close();
}

async function confirmDeleteMultiple() {
  try {
    // Hapus satu per satu ke API
    for (const id of selected.value) {
      await axios.delete(`${DISCOUNT_API_URL}/${id}`);
    }
    // Update state
    discounts.value = discounts.value.filter(d => !selected.value.includes(d._id));
    showToast(`${selected.value.length} diskon berhasil dihapus!`, "success");
    selected.value = [];
  } catch {
    showToast("Gagal menghapus beberapa diskon!", "error");
  } finally {
    closeDeleteMultipleModal();
  }
}


const validateNama = (value) => {
  if (!value || value.trim() === "") return "Nama diskon tidak boleh kosong";
  if (value.trim().length < 3) return "Nama diskon minimal 3 karakter";
  if (
    discounts.value.some(
      (d) =>
        d?.nama_diskon?.toLowerCase() === value.toLowerCase() &&
        d._id !== form.value._id
    )
  ) {
    return "Nama diskon sudah ada";
  }
  return "";
};

const validateJumlah = (value) => {
  if (value === null || value === "" || value === undefined)
    return "Nilai diskon tidak boleh kosong";
  if (value <= 0) return "Nilai diskon harus lebih dari 0";
  if (form.value.type === "percentage" && value > 100)
    return "Persentase tidak boleh lebih dari 100%";
  return "";
};

const formValid = computed(() => {
  nameError.value = validateNama(form.value.nama_diskon);
  jumlahError.value = validateJumlah(form.value.jumlah);
  return !nameError.value && !jumlahError.value;
});

const toasts = ref([]);
function showToast(message, type = "success") {
  const id = Date.now();
  toasts.value.push({ id, message, type });
  setTimeout(() => removeToast(id), 4000);
}
function removeToast(id) {
  toasts.value = toasts.value.filter((t) => t.id !== id);
}
const getLucideIcon = (type) => {
  switch (type) {
    case "success": return CheckCircle;
    case "error": return XCircle;
    case "warning": return AlertTriangle;
    default: return Info;
  }
};

const filteredDiscounts = computed(() => {
  let filtered = discounts.value.filter((d) => {
    const matchStore = !selectedStore.value || d.toko_id === selectedStore.value;
    const matchSearch = d?.nama_diskon?.toLowerCase().includes(search.value.toLowerCase() || "");
    return matchStore && matchSearch;
  });

  // Apply sorting
  if (sortField.value) {
    filtered = [...filtered].sort((a, b) => {
      let aValue = a[sortField.value];
      let bValue = b[sortField.value];

      // For string comparison (nama_diskon)
      if (typeof aValue === 'string') {
        aValue = aValue.toLowerCase();
        bValue = bValue.toLowerCase();
      }

      if (sortOrder.value === 'asc') {
        return aValue > bValue ? 1 : -1;
      } else {
        return aValue < bValue ? 1 : -1;
      }
    });
  }

  return filtered;
});

const totalDiscountCount = computed(() => discounts.value.length);

const paginatedDiscounts = computed(() => filteredDiscounts.value);

const isAllSelected = computed(() =>
  filteredDiscounts.value.length > 0 &&
  selected.value.length === filteredDiscounts.value.length
);
const isIndeterminate = computed(() =>
  selected.value.length > 0 && selected.value.length < filteredDiscounts.value.length
);

function toggleSelectAll(e) {
  if (e.target.checked) {
    selected.value = filteredDiscounts.value.map((d) => d._id);
  } else {
    selected.value = [];
  }
}

function toggleSelect(id) {
  if (selected.value.includes(id)) {
    selected.value = selected.value.filter((s) => s !== id);
  } else {
    selected.value.push(id);
  }
}

function handleSort(field) {
  if (sortField.value === field) {
    sortOrder.value = sortOrder.value === 'asc' ? 'desc' : 'asc';
  } else {
    sortField.value = field;
    sortOrder.value = 'asc';
  }
}

function handleStoreChange() {
  selected.value = [];
}

async function fetchStores() {
  try {
    const res = await axios.get(STORE_API_URL);
    stores.value = res.data;
  } catch {
    showToast("Gagal mengambil data toko!", "error");
  }
}

async function fetchDiscounts() {
  try {
    const res = await axios.get(DISCOUNT_API_URL);
    discounts.value = res.data;
  } catch {
    showToast("Gagal mengambil data diskon!", "error");
  }
}

const modalRef = ref(null);
function openModal() {
  if (!selectedStore.value) {
    showToast("Pilih toko terlebih dahulu di filter!", "warning");
    return;
  }
  editMode.value = false;
  form.value = {
    _id: null,
    nama_diskon: "",
    jumlah: null,
    toko_id: selectedStore.value,
    type: "percentage",
  };
  nameError.value = "";
  jumlahError.value = "";
  modalRef.value?.show();
}

function closeModal() {
  modalRef.value?.close();
}

function editItem(item) {
  editMode.value = true;
  form.value = { ...item };
  nameError.value = "";
  jumlahError.value = "";
  modalRef.value?.show();
}

async function saveItem() {
  if (!formValid.value) {
    showToast("Mohon perbaiki kesalahan pada form!", "error");
    return;
  }
  try {
    const payload = {
      nama_diskon: form.value.nama_diskon.trim(),
      jumlah: form.value.jumlah,
      toko_id: form.value.toko_id,
      type: form.value.type,
    };
    if (editMode.value) {
      await axios.put(`${DISCOUNT_API_URL}/${form.value._id}`, payload);
      const idx = discounts.value.findIndex((d) => d._id === form.value._id);
      if (idx !== -1) discounts.value[idx] = { ...form.value, ...payload };
      showToast(`Diskon "${payload.nama_diskon}" berhasil diperbarui!`, "success");
    } else {
      const res = await axios.post(DISCOUNT_API_URL, payload);
      discounts.value.push(res.data);
      newItemId.value = res.data._id;
      showToast(`Diskon "${payload.nama_diskon}" berhasil ditambahkan!`, "success");
    }
    closeModal();
  } catch {
    showToast("Gagal menyimpan diskon!", "error");
  }
}
onMounted(async () => {
  await fetchStores();
  await fetchDiscounts();
});
</script>

<style scoped>
md-outlined-select {
  --md-outlined-select-text-field-container-shape: 16px;
}

md-outlined-select [slot="leading-icon"] {
  color: inherit !important;
}
</style>