<script lang="ts">
  import axios from "axios";
  import { onMount } from "svelte";

const { onSelectChange, deselectAllItems } = $props() as { onSelectChange: () => void; deselectAllItems: () => void; };

  const SL_API_BASE_URL = "https://24c1tkvfsa.execute-api.ap-southeast-1.amazonaws.com"; // live
  // const SL_API_BASE_URL = "https://z42wre5ee1.execute-api.us-east-1.amazonaws.com"; // dev
  const VERSION_NUMBER = "07.26.24"; // live
  // const VERSION_NUMBER = "2.2.6"; // dev
  const pageSize = 10;

  let token = sessionStorage.getItem("authToken") || "";
    let selectAll = $state(false);

  const state = $state({
    data: [] as any[],
    currentPage: 1,
    totalPages: 1,
    totalItems: 0
  });

  function getCurrentPageItems() {
    return state.data;
  }

  function getSelectedItems() {
    return JSON.parse(sessionStorage.getItem('selectedItems') || '[]');
  }

  function areAllPageItemsSelected() {
    const pageItems = getCurrentPageItems();
    const selectedItems = getSelectedItems();
    
    if (pageItems.length === 0) return false;
    
    return pageItems.every(pageItem => 
      selectedItems.some((selected: any) => selected.listing_id === pageItem.listing_id)
    );
  }

   function toggleSelectAll() {
    const pageItems = getCurrentPageItems();
    const selectedItems = getSelectedItems();
    
    if (selectAll) {
      // Remove all items from current page
      const newSelected = selectedItems.filter((selected: any) => 
        !pageItems.some(pageItem => pageItem.listing_id === selected.listing_id)
      );
      sessionStorage.setItem('selectedItems', JSON.stringify(newSelected));
    } else {
      // Add all items from current page that aren't already selected
      const pageItemIds = pageItems.map(item => item.listing_id);
      const existingIds = new Set(selectedItems.map((item: any) => item.listing_id));
      
      const itemsToAdd = pageItems
        .filter(item => !existingIds.has(item.listing_id))
        .map(item => ({
          listing_id: item.listing_id,
          model: item.model
        }));
      
      const newSelected = [...selectedItems, ...itemsToAdd];
      sessionStorage.setItem('selectedItems', JSON.stringify(newSelected));
    }
    
    // Update UI immediately
    updateSelectAllState();
    
    // Notify parent to update canvas
    onSelectChange();
  }

  function updateSelectAllState() {
    selectAll = areAllPageItemsSelected();
  }

  function selectItem(item: any, checked: boolean) {
    const stored: any[] = getSelectedItems();

    if (checked) {
      if (!stored.find(i => i.listing_id === item.listing_id)) {
        stored.push({
          listing_id: item.listing_id,
          model: item.model
        });
      }
    } else {
      const index = stored.findIndex(i => i.listing_id === item.listing_id);
      if (index > -1) stored.splice(index, 1);
    }

    sessionStorage.setItem('selectedItems', JSON.stringify(stored));

    // Update selectAll state
    updateSelectAllState();
    
    // Notify parent to update canvas
    onSelectChange();

    console.log("Selected items:", stored);
  }

  function clearAllSelections() {
    if (deselectAllItems) {
      deselectAllItems();
    } else {
      // Fallback: just clear session storage
      sessionStorage.removeItem('selectedItems');
      selectAll = false;
      onSelectChange();
    }
  }


  async function fetchData(page = 1) {
    const offsetNumber = (page - 1) * pageSize;

    const inputData = {
      token,
      user_type: 'Xpert',
      owner_type: 'Xpert',
      owner_id: sessionStorage.getItem('xpertId'),
      status: 'Active',
      offset: offsetNumber,
      search: "",
      version_number: VERSION_NUMBER
    };

    console.log(inputData)

    try {
      const res = await axios.post(
        `${SL_API_BASE_URL}/xdeal/GetInventory`,
        inputData
      );

      console.log( `${SL_API_BASE_URL}/xdeal/GetInventory`)
      state.data = (res.data.listed_items || []).map((item: any) => ({
        quantity: item.quantity,
        image_link: item.image_link,
        listing_id: item.listing_id,
        item_no: item.item_no,
        brand: item.brand,
        model: item.model,
        category: item.category,
        appraised_value: item.appraised_value,
        preorder: item.preorder,
        listed: item.listed,
        status: item.status
      }));

      state.totalItems = res.data.listed_items_count || 0;
      state.totalPages = Math.ceil(state.totalItems / pageSize);
      state.currentPage = page;

      console.log("Mapped data:", state.data);
    } catch (error) {
      console.error("Error fetching inventory:", error);
    }
  }

  function goToPage(page: number) {
    if (page < 1 || page > state.totalPages) return;
    fetchData(page);
  }

  onMount(() => {
    fetchData(1);
  });
</script>

<!-- Table component -->
<div class="col-lg-12 col-md-12 mt-4 card p-3 shadow-sm rounded-4 bg-white">
  <!-- Header info -->
  <div class="row justify-content-between d-flex align-items-center mb-3">
    <div class="col-lg-6 col-md-6 d-flex flex-row align-items-center mb-2">
      <div class="w-auto border-end pe-3">
        <label>Showing</label>
        <label class="ms-1 fw-medium">
          {(state.currentPage - 1) * pageSize + 1}-{Math.min(state.currentPage * pageSize, state.totalItems)} of {state.totalItems} items
        </label>
      </div>
      <div class="w-25 ms-3">
        <label>Page</label>
        <label class="ms-1 fw-medium">{state.currentPage} of {state.totalPages}</label>
      </div>
    </div>

    <!-- Select All button -->
    <div class="col-lg-6 col-md-6 d-flex justify-content-end mb-2">
      <button 
        class="btn btn-sm {selectAll ? 'btn-danger' : 'btn-primary'}"
        on:click={toggleSelectAll}
        title={selectAll ? 'Deselect all items on this page' : 'Select all items on this page'}>
        {selectAll ? 'Deselect All (Page)' : 'Select All (Page)'}
      </button>

      <button 
        class="btn btn-sm btn-outline-danger"
        on:click={clearAllSelections}
        title="Clear all selections">
        Clear All
      </button>
    </div>
  </div>

  <hr class="my-3"/>

  <!-- Table -->
  <div class="table-responsive rounded-4 overflow-hidden shadow-sm">
    <table class="table table-striped table-hover text-center align-middle mb-0">
      <thead class="table-dark">
  <tr>
    <th>Select</th>
    <th>QTY</th>
    <th>Item Image</th>
    <th>Reference No</th>
    <th>Brand</th>
    <th>Model</th>
    <th>Category</th>
    <!-- <th>Price</th> -->
    <th>Pre-Order</th>
    <th>Listed</th>
    <th>Status</th>
  </tr>
</thead>

<tbody>
  {#if state.data.length === 0}
    <tr>
      <td colspan="10">No data available</td>
    </tr>
  {:else}
    {#each state.data as row, index}
      <tr>
        <!-- Radio button -->
  <td>
  <input
    type="checkbox"
    checked={getSelectedItems().some((i: any) => i.listing_id === row.listing_id)}
    on:change={(e) => selectItem(row, e.target.checked)} />
</td>

        <td>{row.quantity}</td>
        <td>
          <img src={row.image_link || "https://via.placeholder.com/60"} class="rounded-3 item-image" alt="Item Image" />
        </td>
        <td>{row.listing_id}</td>
        <td>{row.brand}</td>
        <td>{row.model}</td>
        <td>{row.category}</td>
        <!-- <td>${row.appraised_value}</td> -->
        <td>
          <span class="badge {row.preorder ? 'bg-success' : 'bg-danger'}">
            {row.preorder ? 'Yes' : 'No'}
          </span>
        </td>
        <td>
          <span class="badge {row.listed ? 'bg-success' : 'bg-danger'}">
            {row.listed ? 'Yes' : 'No'}
          </span>
        </td>
        <td>
          <span class="badge {row.status === 'Active' ? 'bg-primary' : 'bg-secondary'}">
            {row.status}
          </span>
        </td>
      </tr>
    {/each}
  {/if}
</tbody>

    </table>
  </div>

  <!-- Pagination -->
  <div class="d-flex justify-content-center align-items-center mt-3 gap-2 flex-wrap">
    <button class="btn btn-outline-primary btn-sm rounded-pill" on:click={() => goToPage(state.currentPage - 1)} disabled={state.currentPage === 1}>Prev</button>
    
    {#each Array(state.totalPages) as _, i}
      <button
        class="btn btn-sm rounded-pill {state.currentPage === i+1 ? 'btn-primary' : 'btn-outline-primary'}"
        on:click={() => goToPage(i+1)}>
        {i+1}
      </button>
    {/each}

    <button class="btn btn-outline-primary btn-sm rounded-pill" on:click={() => goToPage(state.currentPage + 1)} disabled={state.currentPage === state.totalPages}>Next</button>
  </div>
</div>

<style>
/* ===== Card ===== */
.btn-sm {
  padding: 6px 16px;
  font-size: 0.85rem;
}

.btn-primary {
  background-color: #3b82f6;
  border-color: #3b82f6;
}

.btn-danger {
  background-color: #ef4444;
  border-color: #ef4444;
}

.btn-primary:hover, .btn-danger:hover {
  opacity: 0.9;
  transform: translateY(-1px);
}

.card {
  background-color: #0f0f0f;
  padding: 24px;
  border-radius: 16px;
  box-shadow: none;
}

.card:hover {
  transform: translateY(-2px);
  box-shadow: 0 14px 30px rgba(0,0,0,0.3);
}

.card hr {
  border-color: #2c2c2c;
}

label {
  color: #cfcfcf;
  font-size: 0.9rem;
}

/* ===== Table Wrapper ===== */
.table-responsive {
  width: 100%;
  max-width: 100%;
  background-color: #141414;
  border-radius: 12px;
  overflow-x: auto;
  box-shadow: none;
}

/* Make table fully stretch */
.table {
  width: 100%;
  table-layout: auto;
}

.table thead {
  background-color: #1f1f1f;
}

.table thead th {
  color: #f1f1f1;
  font-weight: 600;
  font-size: 0.85rem;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  padding: 16px 12px;
  border-bottom: 1px solid #2e2e2e;
}

.table tbody tr {
  background-color: transparent;
  border-bottom: 1px solid #242424;
}

.table tbody tr:hover {
  background-color: rgba(255,255,255,0.06);
}

.table tbody td {
  color: #e6e6e6;
  font-size: 0.95rem;
  padding: 16px 12px;
  vertical-align: middle;
  white-space: nowrap;
}

.table td {
  padding: 14px 10px;
  border: none;
}

/* ===== Images ===== */
.table img.item-image {
  width: 200px;
  height: 200px;
  border-radius: 10px;
  object-fit: cover;
  background-color: #fff;
  padding: 4px;
}

input[type="checkbox"] {
  width: 16px;
  height: 16px;
  accent-color: #3b82f6;
}

/* ===== Badges ===== */
.badge {
  font-size: 0.75rem;
  padding: 6px 12px;
  border-radius: 999px;
  font-weight: 600;
}

/* Yes / No */
.bg-success {
  background-color: rgba(34,197,94,0.18) !important;
  color: #4ade80;
}

.bg-danger {
  background-color: rgba(239,68,68,0.18) !important;
  color: #f87171;
}

/* Status */
.bg-primary {
  background-color: rgba(59,130,246,0.2) !important;
  color: #93c5fd;
}

.bg-secondary {
  background-color: rgba(148,163,184,0.15) !important;
  color: #94a3b8;
}

/* ===== Pagination ===== */
.d-flex button {
  min-width: 38px;
  height: 34px;
  border-radius: 999px;
  font-size: 0.85rem;
  font-weight: 500;
  transition: all 0.15s ease;
}

.btn-outline-primary {
  border-color: #3b82f6;
  color: #3b82f6;
}

.btn-outline-primary:hover:not(:disabled) {
  background-color: rgba(59,130,246,0.15);
  transform: translateY(-1px);
}

.btn-primary {
  background-color: #3b82f6;
  border-color: #3b82f6;
}

.btn:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

</style>

