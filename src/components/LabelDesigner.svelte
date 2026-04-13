<script lang="ts">
  import Dropdown from "bootstrap/js/dist/dropdown";
  import * as fabric from "fabric";
  import { onMount, onDestroy, tick } from "svelte";
  import { QRCode } from "$/fabric-object/qrcode";
  import { ImageEncoder, AbstractPrintTask, Utils, type EncodedImage } from "@mmote/niimbluelib";
import CryptoJS from "crypto-js";
  import { iconCodepoints, type MaterialIcon } from "$/styles/mdi_icons";
  import { automation, connectionState, csvData } from "$/stores";
  import {
    type ExportedLabelTemplate,
    type FabricJson,
    type LabelProps,
    type MoveDirection,
    type OjectType
  } from "$/types";
  import { FileUtils } from "$/utils/file_utils";
  import { tr } from "$/utils/i18n";
  import { LabelDesignerObjectHelper } from "$/utils/label_designer_object_helper";
  import { LocalStoragePersistence } from "$/utils/persistence";
  import { Toasts } from "$/utils/toasts";
  import { UndoRedo, type UndoState } from "$/utils/undo_redo";
  import MdIcon from "$/components/basic/MdIcon.svelte";
  import PrintPreview from "$/components/PrintPreview.svelte";
  import Table from "$/Table.svelte";
  import { DEFAULT_LABEL_PROPS, GRID_SIZE } from "$/defaults";
  import { LabelDesignerUtils } from "$/utils/label_designer_utils";
  import { CustomCanvas } from "$/fabric-object/custom_canvas";
  const XURE_AUCTION_ITEM_LINK = "https://auction.xure.app.xuredeal.com";
  const ENCRYPTION_PASSCODE = "@D4t4dyn4m1xIT";

  // ---------------------------
  // Props
  // ---------------------------
  const { } = $props(); // no incoming props here, just using runes

  // ---------------------------
  // Reactive State
  // ---------------------------
  let htmlCanvas: HTMLCanvasElement;

  let fabricCanvas = $state<CustomCanvas>();
  let labelProps = $state<LabelProps>(DEFAULT_LABEL_PROPS);
  let previewOpened = $state(false);
  let selectedObject = $state<fabric.FabricObject | undefined>(undefined);
  let selectedCount = $state(0);
  let editRevision = $state(0);
  let printNow = $state(false);
  let csvEnabled = $state(false);
  let windowWidth = $state(0);
  let undoState = $state<UndoState>({ undoDisabled: false, redoDisabled: false });
  let itemsToPrint = $state<Array<{listing_id: string; model: string}>>([]);
  let currentPrintIndex = $state(0);

  const undo = new UndoRedo();

  // ---------------------------
  // Methods
  // ---------------------------
  const discardSelection = () => {
    fabricCanvas!.discardActiveObject();
    fabricCanvas!.requestRenderAll();
    selectedObject = undefined;
    selectedCount = 0;
    editRevision = 0;
  };

  const loadLabelData = async (data: ExportedLabelTemplate) => {
    undo.paused = true;
    onUpdateLabelProps(data.label);

    if (data.csv) {
      $csvData = data.csv;
      csvEnabled = true;
    }

    await FileUtils.loadCanvasState(fabricCanvas!, data.canvas);
    undo.paused = false;
  };

  const addHardcodedLabel = () => {
    if (!fabricCanvas) return;

    const selectedItemsRaw = sessionStorage.getItem('selectedItems');
    if (!selectedItemsRaw) return;

    const selectedItems = JSON.parse(selectedItemsRaw);
    const firstItem = selectedItems[0];
    if (!firstItem) return;

    // QR
    const encrypted = CryptoJS.AES
    .encrypt(firstItem.listing_id.toString(), ENCRYPTION_PASSCODE)
    .toString();

  const urlSafe = encrypted
    .replace(/\+/g, "-")
    .replace(/\//g, "_")
    .replace(/=+$/, "");

    const qr = new QRCode({
      text: `${XURE_AUCTION_ITEM_LINK}/xchange_item/${urlSafe}`,
      width: 190,
      height: 190,
      left: 20,
      top: 25
    });
    fabricCanvas.add(qr);

    // Static Text
    const textHello = new fabric.Text("XTORE", {
      left: 220,
      top: 45,
      fontSize: 20,
      fontFamily: "Arial",
      fill: "#000"
    });
    fabricCanvas.add(textHello);

    // Model
    const textName = new fabric.Text(firstItem.model, {
      left: 220,
      top: 75,
      fontSize: 18,
      fontFamily: "Arial",
      fill: "#000"
    });
    fabricCanvas.add(textName);

    fabricCanvas.requestRenderAll();
  };

  const updateCanvasFromSelection = () => {
    if (!fabricCanvas) return;

    fabricCanvas.clear();
    fabricCanvas.backgroundColor = null;

    const storedItems = sessionStorage.getItem('selectedItems');
    if (!storedItems) {
      fabricCanvas.requestRenderAll();
      return;
    }

    const items: { listing_id: string; model: string }[] = JSON.parse(storedItems);
    if (items.length === 0) {
      fabricCanvas.requestRenderAll();
      return;
    }

    let topOffset = 25;
    const spacing = 220;

    

    items.forEach((item, index) => {
      const yPosition = topOffset + index * spacing;

      const encrypted = CryptoJS.AES
    .encrypt(item.listing_id.toString(), ENCRYPTION_PASSCODE)
    .toString();

  const urlSafe = encrypted
    .replace(/\+/g, "-")
    .replace(/\//g, "_")
    .replace(/=+$/, "");


      fabricCanvas.add(new QRCode({
        text: `${XURE_AUCTION_ITEM_LINK}/xchange_item/${urlSafe}`,
        width: 190,
        height: 190,
        left: 20,
        top: yPosition
      }));

      fabricCanvas.add(new fabric.Text("XTORE", {
        left: 220,
        top: yPosition + 20,
        fontSize: 20,
        fontFamily: "Arial",
        fill: "#000"
      }));

      fabricCanvas.add(new fabric.Text(item.model, {
        left: 220,
        top: yPosition + 50,
        fontSize: 18,
        fontFamily: "Arial",
        fill: "#000"
      }));
    });

    fabricCanvas.requestRenderAll();
  };

  const deleteSelected = () => {
    LabelDesignerUtils.deleteSelection(fabricCanvas!);
    discardSelection();
  };

  const cloneSelected = () => {
    LabelDesignerUtils.cloneSelection(fabricCanvas!).then(() => undo.push(fabricCanvas!, labelProps));
  };

  const moveSelected = (direction: MoveDirection, ctrl?: boolean) => {
    LabelDesignerUtils.moveSelection(fabricCanvas!, direction, ctrl);
    undo.push(fabricCanvas!, labelProps);
  };

  const onKeyDown = (e: KeyboardEvent) => {
    const key = e.key.toLowerCase();
    const cmdOrCtrl = e.metaKey || e.ctrlKey;

    if (key === "escape") {
      discardSelection();
      return;
    }

    if (LabelDesignerUtils.isAnyInputFocused(fabricCanvas!)) return;

    if (key.startsWith("arrow")) {
      moveSelected(key.slice("arrow".length) as MoveDirection, cmdOrCtrl);
      return;
    }

    if (e.repeat) return;

    if (cmdOrCtrl && key === "d") {
      e.preventDefault();
      cloneSelected();
      return;
    }

    if ((cmdOrCtrl && key === "y") || (cmdOrCtrl && e.shiftKey && key === "z")) {
      e.preventDefault();
      if (!undoState.redoDisabled) undo.redo();
      return;
    }

    if (cmdOrCtrl && key === "z") {
      e.preventDefault();
      if (!undoState.undoDisabled) undo.undo();
      return;
    }

    if (key === "delete" || key === "backspace") {
      deleteSelected();
    }
  };

  const onUpdateLabelProps = (newProps: LabelProps) => {
    labelProps = newProps;
    fabricCanvas!.setDimensions(labelProps.size);
    fabricCanvas!.virtualZoom(fabricCanvas!.getVirtualZoom());

    try {
      LocalStoragePersistence.saveLastLabelProps(labelProps);
      undo.push(fabricCanvas!, labelProps);
    } catch (e) {
      Toasts.zodErrors(e, "Label parameters save error:");
    }
  };

  const deselectAllItems = () => {
    // Clear session storage
    sessionStorage.removeItem('selectedItems');
    
    // Clear the canvas
    if (fabricCanvas) {
      fabricCanvas.clear();
      fabricCanvas.backgroundColor = null;
      fabricCanvas.requestRenderAll();
    }
    
    // Reset any state related to selections
    selectedObject = undefined;
    selectedCount = 0;
    
    // Trigger a re-render of the table if needed
    updateCanvasFromSelection();
  };

  const onPreviewClosed = () => {
    printNow = false;
    previewOpened = false;

    if (itemsToPrint.length > 0) {
      currentPrintIndex++;
      tick().then(() => {
        printNextItem();
      });
    } else {
      // All printing is done, deselect all items
      setTimeout(() => {
        deselectAllItems();
      }, 100); // Small delay to ensure everything is cleaned up
    }
  };

  const openPreviewAndPrint = () => {
    const storedItems = sessionStorage.getItem('selectedItems');
    if (!storedItems || JSON.parse(storedItems).length === 0) return;

    const items = JSON.parse(storedItems);
    if (items.length === 0) {
      alert("No items selected to print");
      return;
    }
    itemsToPrint = items;
    currentPrintIndex = 0;
    printNextItem();
    // printNow = true;
    // previewOpened = true;
  };

  // New function to handle sequential printing
  const printNextItem = () => {
    if (currentPrintIndex >= itemsToPrint.length) {
      let printedCount = itemsToPrint.length;

      // Clear session storage after printing all items
      deselectAllItems();

      // All done!
      itemsToPrint = [];
      currentPrintIndex = 0;
      alert(`Printed ${printedCount} labels.`);
      return;
    }

    // Generate canvas for current item only
    fabricCanvas!.clear();
    const item = itemsToPrint[currentPrintIndex];

    const encrypted = CryptoJS.AES
    .encrypt(item.listing_id.toString(), ENCRYPTION_PASSCODE)
    .toString();

  const urlSafe = encrypted
    .replace(/\+/g, "-")
    .replace(/\//g, "_")
    .replace(/=+$/, "");
    
    fabricCanvas!.add(new QRCode({
      text: `${XURE_AUCTION_ITEM_LINK}/xchange_item/${urlSafe}`,
      width: 190,
      height: 190,
      left: 20,
      top: 25
    }));

    fabricCanvas!.add(new fabric.Text("XTORE", {
      left: 220,
      top: 45,
      fontSize: 20,
      fontFamily: "Arial",
      fill: "#000"
    }));

    fabricCanvas!.add(new fabric.Text(item.model, {
      left: 220,
      top: 75,
      fontSize: 18,
      fontFamily: "Arial",
      fill: "#000"
    }));

    fabricCanvas!.requestRenderAll();
    
    // Open preview for this single item
    printNow = true;
    previewOpened = true;
  };



  const getCanvasForPreview = (): FabricJson => fabricCanvas!.toJSON();

  const onPaste = async (event: ClipboardEvent) => {
    if (LabelDesignerUtils.isAnyInputFocused(fabricCanvas!)) return;
    if (document.querySelectorAll(".dropdown-menu.show").length > 0) return;

    if (event.clipboardData) {
      event.preventDefault();
      const obj = await LabelDesignerObjectHelper.addObjectFromClipboard(fabricCanvas!, event.clipboardData);
      if (obj) {
        fabricCanvas!.setActiveObject(obj);
        undo.push(fabricCanvas!, labelProps);
      }
    }
  };

  // ---------------------------
  // Lifecycle
  // ---------------------------
onMount(() => {
  // Initialize canvas
  fabricCanvas = new CustomCanvas(htmlCanvas, {
    width: labelProps.size.width,
    height: labelProps.size.height
  });
  fabricCanvas.setLabelProps(labelProps);

  // Push initial undo state
  undo.push(fabricCanvas, labelProps);

  // Render QR/text for selected items (if any)
  updateCanvasFromSelection(); // <--- FIX: this ensures canvas matches sessionStorage
});

  onDestroy(() => {
    fabricCanvas?.dispose();
  });

  $effect(() => {
    fabricCanvas?.setLabelProps(labelProps);
  });

  undo.onLabelUpdate = loadLabelData;
  undo.onStateUpdate = (state: UndoState) => undoState = state;
</script>

<svelte:window bind:innerWidth={windowWidth} onkeydown={onKeyDown} onpaste={onPaste} />

<div class="image-editor">
  <div class="row mb-3">
    <div class="col d-flex {windowWidth === 0 || labelProps.size.width < windowWidth ? 'justify-content-center' : ''}">
      <div class="canvas-wrapper print-start-{labelProps.printDirection}">
        <canvas bind:this={htmlCanvas}></canvas>
      </div>
    </div>
  </div>

  <div class="row mb-1">
    <div class="col d-flex justify-content-center">
      <div class="toolbar d-flex flex-wrap gap-1 justify-content-center align-items-center">
        <button
          title="Print with default or saved parameters"
          class="btn btn-sm btn-primary ms-1"
          on:click={openPreviewAndPrint}
          disabled={$connectionState !== "connected"}>
          <MdIcon icon="print" /> {$tr("editor.print")}
        </button>
      </div>
    </div>
  </div>

  {#if previewOpened}
    <PrintPreview
      key={Date.now()}
      onClosed={onPreviewClosed}
      canvasCallback={getCanvasForPreview}
      {labelProps}
      {printNow}
      {csvEnabled}
      csvData={$csvData.data} />
  {/if}

  <Table onSelectChange={updateCanvasFromSelection} deselectAllItems={deselectAllItems} />
</div>

<style>
.canvas-wrapper {
  border: 1px solid rgba(0, 0, 0, 0.4);
  background-color: rgba(60, 55, 63, 0.5);
}
.canvas-wrapper.print-start-left {
  border-left: 2px solid #ff4646;
}
.canvas-wrapper.print-start-top {
  border-top: 2px solid #ff4646;
}
.canvas-wrapper canvas {
  image-rendering: pixelated;
}
</style>
