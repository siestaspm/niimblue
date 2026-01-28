<script lang="ts">
  import axios from "axios";
  import { onMount } from "svelte";

const { onSelectChange } = $props() as { onSelectChange: () => void };

  const SL_API_BASE_URL = "https://z42wre5ee1.execute-api.us-east-1.amazonaws.com";
  const VERSION_NUMBER = "2.2.6";
  const pageSize = 10;

  let token = sessionStorage.getItem("authToken") || "";

  const state = $state({
    data: [] as any[],
    currentPage: 1,
    totalPages: 1,
    totalItems: 0
  });

function selectItem(item: any, checked: boolean) {
  // Get current array from sessionStorage
  const stored: any[] = JSON.parse(sessionStorage.getItem('selectedItems') || '[]');

  if (checked) {
    // Add item if it's not already in the array
    if (!stored.find(i => i.listing_id === item.listing_id)) {
      stored.push({
        listing_id: item.listing_id,
        model: item.model
      });
    }
  } else {
    // Remove item if unchecked
    const index = stored.findIndex(i => i.listing_id === item.listing_id);
    if (index > -1) stored.splice(index, 1);
  }

  onSelectChange()

  sessionStorage.setItem('selectedItems', JSON.stringify(stored));
  console.log("Selected items:", stored);
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
    <th>Price</th>
    <th>Pre-Order</th>
    <th>Listed</th>
    <th>Status</th>
  </tr>
</thead>

<tbody>
  {#if state.data.length === 0}
    <tr>
      <td colspan="11">No data available</td>
    </tr>
  {:else}
    {#each state.data as row, index}
      <tr>
        <!-- Radio button -->
  <td>
  <input
    type="checkbox"
    checked={JSON.parse(sessionStorage.getItem('selectedItems') || '[]').some(i => i.listing_id === row.listing_id)}
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
        <td>${row.appraised_value}</td>
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
.card {
  transition: all 0.2s;
  
}

.card:hover {
  transform: translateY(-3px);
  box-shadow: 0 8px 20px rgba(0,0,0,0.1);
}

/* Table styles */
.table-responsive {
  background-color: #5b5b5b; /* black background */
  border-radius: 16px;
  overflow: hidden;
  box-shadow: 0 4px 10px rgba(0,0,0,0.1);
}

.table {
  color: #fff; /* white text */
}

.table thead {
  background-color: #111; /* darker header */
}

.table thead th {
  color: #fff;
  border-bottom: 2px solid #333;
}

.table tbody tr {
  border-bottom: 1px solid #222;
}

.table-hover tbody tr:hover {
  background-color: rgba(255, 255, 255, 0.05);
  transition: 0.2s;
}

.table img.item-image {
  width: 60px;
  height: 60px;
  object-fit: cover;
  border-radius: 8px;
}

/* Badges */
.badge {
  font-size: 0.85rem;
  padding: 0.45em 0.7em;
  border-radius: 12px;
}

/* Pagination buttons */
.d-flex button {
  min-width: 40px;
  transition: all 0.2s;
}

.d-flex button:hover:not(:disabled) {
  transform: scale(1.1);
}

.btn-sm {
  border-radius: 20px;
  padding: 0.35rem 0.7rem;
  font-weight: 500;
}

/* Card and general layout */
.card {
  background-color: #111; /* dark card */
  color: #fff;
}
.card hr {
  border-top: 1px solid #333;
}

label {
  color: #fff;
}
</style>

