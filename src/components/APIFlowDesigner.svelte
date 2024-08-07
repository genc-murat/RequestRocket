<script lang="ts">
  import { onMount, createEventDispatcher } from "svelte";
  import { writable, derived, get } from "svelte/store";
  import type { Writable, Readable } from "svelte/store";
  import { v4 as uuidv4 } from "uuid";
  import { fade, fly } from "svelte/transition";
  import type {
    Flow,
    FlowBlock,
    Connection,
    BlockType,
    SwitchCase,
  } from "../types/flow-types";
  import { blockTypes } from "../types/flow-types";
  import "./APIFlowDesignerStyles.css";

  export let initialFlow: Flow | null = null;
  export let readOnly: boolean = false;

  const dispatch = createEventDispatcher();

  let flow: Writable<Flow> = writable(
    initialFlow || {
      id: uuidv4(),
      name: "New Flow",
      description: "",
      version: "1.0.0",
      blocks: [],
      connections: [],
      variables: {},
      createdAt: new Date(),
      updatedAt: new Date(),
      createdBy: "Anonymous",
      tags: [],
    }
  );

  let selectedBlock: Writable<FlowBlock | null> = writable(null);
  let isConnecting = false;
  let connectionStart: string | null = null;
  let connectionType: "next" | "alternative" | "error" = "next";
  let connectionEnd: { x: number; y: number } | null = null;
  let isDragging: boolean = false;
  let dragOffset: { x: number; y: number } | null = null;
  let showPropertiesModal: boolean = false;
  let zoom: number = 1;
  let pan: { x: number; y: number } = { x: 0, y: 0 };
  let searchTerm: Writable<string> = writable("");
  let categoryFilter: Writable<string> = writable("all");
  let showVariablesPanel = false;
  let showHelpPanel = false;
  let showContextMenu = false;
  let contextMenuPosition = { x: 0, y: 0 };
  let contextMenuTargetId: string | null = null;
  let invalidConnectionMessage: Writable<string> = writable("");

  const blockCategories = {
    all: "All",
    "input-output": "Input/Output",
    logic: "Logic",
    data: "Data Manipulation",
    util: "Utilities",
  };

  type BlockType = keyof typeof blockTypes;

  function isValidBlockType(type: string): type is BlockType {
    return type in blockTypes;
  }

  const blockColors: Record<BlockType, string> = {
    api_call: "#4CAF50",
    condition: "#2196F3",
    loop: "#FF9800",
    timer: "#9C27B0",
    variable: "#E91E63",
    json_transformer: "#00BCD4",
    regex: "#795548",
    webhook: "#3F51B5",
    custom_script: "#607D8B",
    try_catch: "#F44336",
    switch_case: "#FFEB3B",
    auth: "#8BC34A",
    comment: "#9E9E9E",
    group: "#FFC107",
    start: "#4CAF50", // Başlangıç için yeşil
    end: "#F44336", // Bitiş için kırmızı
  };

  const filteredBlocks: Readable<FlowBlock[]> = derived(
    [flow, searchTerm, categoryFilter],
    ([$flow, $searchTerm, $categoryFilter]) => {
      return $flow.blocks.filter(
        (block) =>
          block.type.toLowerCase().includes($searchTerm.toLowerCase()) &&
          ($categoryFilter === "all" ||
            blockTypes[block.type].category === $categoryFilter)
      );
    }
  );

  // Undo/Redo işlemleri için geçmiş
  let history: Flow[] = [];
  let historyIndex = -1;

  function saveToHistory() {
    history = history.slice(0, historyIndex + 1);
    history.push(JSON.parse(JSON.stringify(get(flow))));
    historyIndex++;
  }

  function undo() {
    if (historyIndex > 0) {
      historyIndex--;
      flow.set(JSON.parse(JSON.stringify(history[historyIndex])));
    }
  }

  function redo() {
    if (historyIndex < history.length - 1) {
      historyIndex++;
      flow.set(JSON.parse(JSON.stringify(history[historyIndex])));
    }
  }

  onMount(() => {
    const canvas = document.querySelector(".canvas") as HTMLElement;
    if (canvas) {
      canvas.addEventListener("wheel", handleZoom);
      canvas.addEventListener("mousedown", startPan);
      canvas.addEventListener("mousemove", handlePan);
      canvas.addEventListener("mouseup", endPan);
      canvas.addEventListener("mouseleave", endPan);
    }
    document.addEventListener("keydown", handleKeyDown);
    saveToHistory();

    return () => {
      document.removeEventListener("keydown", handleKeyDown);
    };
  });

  function addBlock(type: string) {
    if (isValidBlockType(type)) {
      const newBlock: FlowBlock = {
        id: uuidv4(),
        type,
        data: {},
        position: { x: 100, y: 100 },
        next: null,
      };
      flow.update((f) => ({
        ...f,
        blocks: [...f.blocks, newBlock],
      }));
      saveToHistory();
    } else {
      console.error(`Invalid block type: ${type}`);
    }
  }

  function updateBlockPosition(id: string, position: { x: number; y: number }) {
    flow.update((f) => ({
      ...f,
      blocks: f.blocks.map((block) =>
        block.id === id ? { ...block, position } : block
      ),
    }));
    saveToHistory();
  }

  function connectBlocks(
    sourceId: string,
    targetId: string,
    type: Connection["type"] = "next"
  ) {
    flow.update((f) => ({
      ...f,
      connections: [
        ...f.connections,
        { id: uuidv4(), source: sourceId, target: targetId, type },
      ],
    }));
    saveToHistory();
  }

  function deleteBlock(id: string) {
    flow.update((f) => ({
      ...f,
      blocks: f.blocks.filter((block) => block.id !== id),
      connections: f.connections.filter(
        (conn) => conn.source !== id && conn.target !== id
      ),
    }));
    saveToHistory();
  }

  function deleteConnection(id: string) {
    flow.update((f) => ({
      ...f,
      connections: f.connections.filter((conn) => conn.id !== id),
    }));
    saveToHistory();
  }

  function saveFlow() {
    dispatch("save", get(flow));
  }

  function runFlow() {
    dispatch("run", get(flow));
  }

  function handleDragStart(event: DragEvent, blockId: string) {
    if (readOnly || !event.target) return;
    isDragging = true;
    const target = event.target as HTMLElement;
    const rect = target.getBoundingClientRect();
    dragOffset = {
      x: event.clientX - rect.left,
      y: event.clientY - rect.top,
    };
    event.dataTransfer?.setData("text/plain", blockId);
    target.classList.add("dragging"); // Sürükleme sırasında sınıf ekle
  }

  function handleDragEnd(event: DragEvent) {
    isDragging = false;
    dragOffset = null;
    const target = event.target as HTMLElement;
    target.classList.remove("dragging"); // Sürükleme sona erdiğinde sınıfı kaldır
  }

  function handleDrop(event: DragEvent) {
    event.preventDefault();
    const blockId = event.dataTransfer?.getData("text/plain");
    if (blockId && dragOffset) {
      const canvas = event.currentTarget as HTMLElement;
      const canvasRect = canvas.getBoundingClientRect();
      const newPosition = {
        x: (event.clientX - canvasRect.left - dragOffset.x - pan.x) / zoom,
        y: (event.clientY - canvasRect.top - dragOffset.y - pan.y) / zoom,
      };
      updateBlockPosition(blockId, newPosition);
    }
    isDragging = false;
    dragOffset = null;
  }

  let highlightedBlockId: string | null = null;

  function startConnection(blockId: string, event: MouseEvent) {
    isConnecting = true;
    connectionStart = blockId;
    connectionEnd = { x: event.clientX, y: event.clientY };
    event.stopPropagation();
  }

  function handleMouseMove(event: MouseEvent) {
    if (isConnecting) {
      const canvas = document.querySelector(".canvas") as HTMLElement;
      if (canvas) {
        const canvasRect = canvas.getBoundingClientRect();
        connectionEnd = {
          x: (event.clientX - canvasRect.left - pan.x) / zoom,
          y: (event.clientY - canvasRect.top - pan.y) / zoom,
        };
      }

      const targetElement = event.target as HTMLElement;
      const targetBlock = targetElement.closest(".block") as HTMLElement;

      if (targetBlock && targetBlock.dataset.id !== highlightedBlockId) {
        if (highlightedBlockId) {
          document
            .querySelector(`[data-id="${highlightedBlockId}"]`)
            ?.classList.remove("highlight");
        }
        highlightedBlockId = targetBlock.dataset.id || null;
        targetBlock.classList.add("highlight");
      } else if (!targetBlock && highlightedBlockId) {
        document
          .querySelector(`[data-id="${highlightedBlockId}"]`)
          ?.classList.remove("highlight");
        highlightedBlockId = null;
      }
    }
  }

  function handleMouseUp(event: MouseEvent) {
    if (isConnecting && connectionStart) {
      const targetElement = event.target as HTMLElement;
      const targetBlock = targetElement.closest(".block") as HTMLElement;
      if (targetBlock) {
        const targetId = targetBlock.dataset.id;
        if (
          targetId &&
          targetId !== connectionStart &&
          isConnectionValid(connectionStart, targetId)
        ) {
          connectBlocks(connectionStart, targetId, connectionType);
        } else {
          showInvalidConnectionFeedback();
        }
      }
    }
    isConnecting = false;
    connectionStart = null;
    connectionEnd = null;
    if (highlightedBlockId) {
      document
        .querySelector(`[data-id="${highlightedBlockId}"]`)
        ?.classList.remove("highlight");
      highlightedBlockId = null;
    }
  }

  function isConnectionValid(sourceId: string, targetId: string): boolean {
    const currentFlow = get(flow);
    const sourceBlock = currentFlow.blocks.find((b) => b.id === sourceId);
    const targetBlock = currentFlow.blocks.find((b) => b.id === targetId);

    if (!sourceBlock || !targetBlock) {
      invalidConnectionMessage.set("Geçersiz blok seçimi.");
      return false;
    }

    if (isCircularConnection(sourceId, targetId)) {
      invalidConnectionMessage.set("Döngüsel bağlantılar oluşturulamaz.");
      return false;
    }

    const existingConnection = currentFlow.connections.find(
      (conn) => conn.source === sourceId && conn.type === connectionType
    );
    if (existingConnection) {
      invalidConnectionMessage.set(
        `Bu bloktan zaten bir ${connectionType} bağlantısı mevcut.`
      );
      return false;
    }

    if (sourceBlock.type === "condition" && connectionType === "next") {
      invalidConnectionMessage.set(
        'Koşul bloğundan "next" bağlantısı oluşturulamaz. "alternative" veya "error" kullanın.'
      );
      return false;
    }

    if (sourceBlock.data.isStartBlock || targetBlock.data.isEndBlock) {
      invalidConnectionMessage.set(
        "Başlangıç bloğuna giriş veya bitiş bloğundan çıkış bağlantısı oluşturulamaz."
      );
      return false;
    }

    return true;
  }

  function isCircularConnection(sourceId: string, targetId: string): boolean {
    const currentFlow = get(flow);

    function checkCircular(
      currentId: string,
      visited: Set<string> = new Set()
    ): boolean {
      if (visited.has(currentId)) return false;
      if (currentId === targetId) return true;

      visited.add(currentId);
      const nextConnections = currentFlow.connections.filter(
        (conn) => conn.source === currentId
      );

      for (const conn of nextConnections) {
        if (checkCircular(conn.target, new Set(visited))) return true;
      }

      return false;
    }

    return checkCircular(sourceId);
  }

  function showInvalidConnectionFeedback() {
    const message = get(invalidConnectionMessage);

    const toast = document.createElement("div");
    toast.className = "toast error";
    toast.textContent = message;
    document.body.appendChild(toast);

    setTimeout(() => {
      toast.classList.add("show");
      setTimeout(() => {
        toast.classList.remove("show");
        setTimeout(() => toast.remove(), 300);
      }, 3000);
    }, 100);

    const invalidLine = document.createElement("div");
    invalidLine.className = "invalid-connection-line";
    document.querySelector(".canvas")?.appendChild(invalidLine);

    const sourceBlock = document.querySelector(
      `[data-id="${connectionStart}"]`
    );
    const targetBlock = document.querySelector(`[data-id="${connectionEnd}"]`);

    if (sourceBlock && targetBlock) {
      const sourceRect = sourceBlock.getBoundingClientRect();
      const targetRect = targetBlock.getBoundingClientRect();

      invalidLine.style.left = `${sourceRect.left + sourceRect.width / 2}px`;
      invalidLine.style.top = `${sourceRect.top + sourceRect.height / 2}px`;
      invalidLine.style.width = `${targetRect.left - sourceRect.left}px`;
      invalidLine.style.height = `${targetRect.top - sourceRect.top}px`;
    }

    setTimeout(() => invalidLine.remove(), 1000);
  }

  function openPropertiesModal(block: FlowBlock) {
    selectedBlock.set(block);
    showPropertiesModal = true;
  }

  function closePropertiesModal() {
    showPropertiesModal = false;
    selectedBlock.set(null);
  }

  function updateBlockProperties() {
    if (get(selectedBlock)) {
      flow.update((f) => ({
        ...f,
        blocks: f.blocks.map((block) =>
          block.id === get(selectedBlock)!.id ? get(selectedBlock)! : block
        ),
      }));
    }
    closePropertiesModal();
    saveToHistory();
  }

  function handleZoom(event: WheelEvent) {
    event.preventDefault();
    const zoomSpeed = 0.1;
    const zoomDelta = event.deltaY > 0 ? -zoomSpeed : zoomSpeed;
    zoom = Math.max(0.1, Math.min(2, zoom + zoomDelta));
  }

  let isPanning = false;
  let panStart = { x: 0, y: 0 };

  function startPan(event: MouseEvent) {
    if (event.button === 1) {
      isPanning = true;
      panStart = { x: event.clientX, y: event.clientY };
    }
  }

  function handlePan(event: MouseEvent) {
    if (isPanning) {
      const dx = event.clientX - panStart.x;
      const dy = event.clientY - panStart.y;
      pan = { x: pan.x + dx, y: pan.y + dy };
      panStart = { x: event.clientX, y: event.clientY };
    }
  }

  function endPan() {
    isPanning = false;
  }

  function handleKeyDown(event: KeyboardEvent) {
    if (event.ctrlKey || event.metaKey) {
      switch (event.key) {
        case "z":
          event.preventDefault();
          undo();
          break;
        case "y":
          event.preventDefault();
          redo();
          break;
        case "s":
          event.preventDefault();
          saveFlow();
          break;
        case "f":
          event.preventDefault();
          (
            document.querySelector('input[type="text"]') as HTMLInputElement
          )?.focus();
          break;
        case "g":
          event.preventDefault();
          autoArrangeBlocks();
          break;
      }
    }
  }

  function openContextMenu(event: MouseEvent, blockId: string) {
    event.preventDefault();
    contextMenuPosition = { x: event.clientX, y: event.clientY };
    contextMenuTargetId = blockId;
    showContextMenu = true;
  }

  function closeContextMenu() {
    showContextMenu = false;
    contextMenuTargetId = null;
  }

  function duplicateBlock() {
    if (contextMenuTargetId) {
      const originalBlock = get(flow).blocks.find(
        (b) => b.id === contextMenuTargetId
      );
      if (originalBlock) {
        const newBlock: FlowBlock = {
          ...JSON.parse(JSON.stringify(originalBlock)),
          id: uuidv4(),
          position: {
            x: originalBlock.position.x + 20,
            y: originalBlock.position.y + 20,
          },
        };
        flow.update((f) => ({
          ...f,
          blocks: [...f.blocks, newBlock],
        }));
        saveToHistory();
      }
    }
    closeContextMenu();
  }

  /**
 * 
 * Her akışın bir "start" bloğu olmalıdır.
Her akışın en az bir "end" bloğu olmalıdır.
Koşul bloklarının "alternative" ve "error" bağlantıları olmalıdır.
Döngü bloklarının maksimum iterasyon sayısı belirtilmelidir.
Her API çağrısı bloğunda URL belirtilmelidir.
JSON dönüştürücü bloklarında dönüştürme fonksiyonu belirtilmelidir.
Her bloğun yalnızca bir türde bağlantısı olmalıdır ("next", "alternative" veya "error").
Bloklar döngüsel bağlantılara sahip olmamalıdır.
Blokların bir grup içerisinde olup olmadığı kontrol edilmelidir (opsiyonel).
Her koşul bloğunun "true" ve "false" dalları olmalıdır.
 */

  function validateFlow() {
    const currentFlow = get(flow);
    const unconnectedBlocks = currentFlow.blocks.filter(
      (block) =>
        !currentFlow.connections.some(
          (conn) => conn.source === block.id || conn.target === block.id
        )
    );

    if (unconnectedBlocks.length > 0) {
      console.warn("Unconnected blocks found:", unconnectedBlocks);
    }

    // Her akışın bir "start" bloğu olmalı
    const startBlocks = currentFlow.blocks.filter(
      (block) => block.type === "start"
    );
    if (startBlocks.length === 0) {
      console.warn("Flow must have at least one start block.");
    }

    // Her akışın en az bir "end" bloğu olmalı
    const endBlocks = currentFlow.blocks.filter(
      (block) => block.type === "end"
    );
    if (endBlocks.length === 0) {
      console.warn("Flow must have at least one end block.");
    }

    // Koşul bloklarının "alternative" ve "error" bağlantıları olmalı
    const conditionBlocks = currentFlow.blocks.filter(
      (block) => block.type === "condition"
    );
    conditionBlocks.forEach((block) => {
      const hasAlternativeConnection = currentFlow.connections.some(
        (conn) => conn.source === block.id && conn.type === "alternative"
      );
      const hasErrorConnection = currentFlow.connections.some(
        (conn) => conn.source === block.id && conn.type === "error"
      );
      const hasNextConnection = currentFlow.connections.some(
        (conn) => conn.source === block.id && conn.type === "next"
      );

      if (!hasAlternativeConnection) {
        console.warn(
          `Condition block ${block.id} is missing an alternative connection.`
        );
      }
      if (!hasErrorConnection) {
        console.warn(
          `Condition block ${block.id} is missing an error connection.`
        );
      }
      if (!hasNextConnection) {
        console.warn(
          `Condition block ${block.id} is missing a next connection.`
        );
      }
    });

    // Döngü bloklarının maksimum iterasyon sayısı belirtilmeli
    const loopBlocks = currentFlow.blocks.filter(
      (block) => block.type === "loop"
    );
    loopBlocks.forEach((block) => {
      if (!block.data.maxIterations || block.data.maxIterations <= 0) {
        console.warn(
          `Loop block ${block.id} must have a valid maxIterations value.`
        );
      }
    });

    // Her API çağrısı bloğunda URL belirtilmeli
    const apiCallBlocks = currentFlow.blocks.filter(
      (block) => block.type === "api_call"
    );
    apiCallBlocks.forEach((block) => {
      if (!block.data.url) {
        console.warn(`API Call block ${block.id} must have a URL.`);
      }
    });

    // JSON dönüştürücü bloklarında dönüştürme fonksiyonu belirtilmeli
    const jsonTransformerBlocks = currentFlow.blocks.filter(
      (block) => block.type === "json_transformer"
    );
    jsonTransformerBlocks.forEach((block) => {
      if (!block.data.transformFunction) {
        console.warn(
          `JSON Transformer block ${block.id} must have a transform function.`
        );
      }
    });

    // Her bloğun yalnızca bir türde bağlantısı olmalı ("next", "alternative" veya "error")
    currentFlow.blocks.forEach((block) => {
      const connections = currentFlow.connections.filter(
        (conn) => conn.source === block.id
      );
      const connectionTypes = connections.map((conn) => conn.type);
      const uniqueConnectionTypes = new Set(connectionTypes);

      if (uniqueConnectionTypes.size !== connectionTypes.length) {
        console.warn(
          `Block ${block.id} has multiple connections of the same type.`
        );
      }
    });

    // Bloklar döngüsel bağlantılara sahip olmamalı
    currentFlow.connections.forEach((connection) => {
      if (isCircularConnection(connection.source, connection.target)) {
        console.warn(
          `Connection from ${connection.source} to ${connection.target} creates a circular dependency.`
        );
      }
    });

    // Blokların bir grup içerisinde olup olmadığı kontrol edilmelidir (opsiyonel)
    currentFlow.blocks.forEach((block) => {
      if (block.type === "group" && !block.group) {
        console.warn(`Group block ${block.id} must be part of a group.`);
      }
    });

    // Her koşul bloğunun "true" ve "false" dalları olmalıdır
    conditionBlocks.forEach((block) => {
      const trueBranch = currentFlow.connections.find(
        (conn) => conn.source === block.id && conn.type === "next"
      );
      const falseBranch = currentFlow.connections.find(
        (conn) => conn.source === block.id && conn.type === "alternative"
      );

      if (!trueBranch || !falseBranch) {
        console.warn(
          `Condition block ${block.id} must have both true and false branches.`
        );
      }
    });

    // Daha fazla doğrulama kuralı buraya eklenebilir...
  }

  function generatePath(
    start: { x: number; y: number },
    end: { x: number; y: number }
  ): string {
    const midX = (start.x + end.x) / 2;
    return `M ${start.x} ${start.y} C ${midX} ${start.y}, ${midX} ${end.y}, ${end.x} ${end.y}`;
  }

  function addVariable() {
    flow.update((f) => ({
      ...f,
      variables: { ...f.variables, ["newVariable"]: "" },
    }));
  }

  function deleteVariable(key: string) {
    flow.update((f) => {
      const { [key]: _, ...rest } = f.variables;
      return { ...f, variables: rest };
    });
  }

  function autoArrangeBlocks() {
    let x = 100;
    let y = 100;
    flow.update((f) => {
      f.blocks = f.blocks.map((block, index) => {
        block.position = { x, y };
        x += 200;
        if ((index + 1) % 5 === 0) {
          x = 100;
          y += 150;
        }
        return block;
      });
      return f;
    });
    saveToHistory();
  }

  function selectConnectionType(type: "next" | "alternative" | "error") {
    connectionType = type;
  }
</script>

<svelte:window on:keydown={handleKeyDown} on:click={closeContextMenu} />

<div class="api-flow-designer">
  <div class="toolbar">
    <input
      type="text"
      bind:value={$searchTerm}
      placeholder="Search blocks..."
    />
    <select bind:value={$categoryFilter} class="w-full">
      {#each Object.entries(blockCategories) as [value, name]}
        <option {value}>{name}</option>
      {/each}
    </select>
    {#each Object.entries(blockTypes) as [type, { category, name }]}
      {#if $categoryFilter === "all" || $categoryFilter === category}
        <button on:click={() => addBlock(type)}>{name}</button>
      {/if}
    {/each}
    <button on:click={saveFlow}>Save Flow</button>
    <button on:click={runFlow}>Run Flow</button>
    <button on:click={validateFlow}>Validate Flow</button>
    <button on:click={() => (showVariablesPanel = true)}>Variables</button>
    <button on:click={() => (showHelpPanel = true)}>Help</button>
    <button on:click={undo} disabled={historyIndex <= 0}>Undo</button>
    <button on:click={redo} disabled={historyIndex >= history.length - 1}
      >Redo</button
    >
    <button on:click={autoArrangeBlocks}>Auto Arrange</button>
  </div>

  <div
    class="canvas"
    on:dragover|preventDefault
    on:drop={handleDrop}
    on:dragend={handleDragEnd}
    on:mousemove={handleMouseMove}
    on:mouseup={handleMouseUp}
    style="transform: scale({zoom}) translate({pan.x}px, {pan.y}px);"
  >
    <svg class="connections" width="100%" height="100%">
      {#each $flow.connections as connection}
        {@const sourceBlock = $flow.blocks.find(
          (b) => b.id === connection.source
        )}
        {@const targetBlock = $flow.blocks.find(
          (b) => b.id === connection.target
        )}
        {#if sourceBlock && targetBlock}
          <path
            d={generatePath(sourceBlock.position, targetBlock.position)}
            fill="none"
            stroke={connection.type === "next"
              ? "green"
              : connection.type === "alternative"
                ? "blue"
                : "red"}
            stroke-width="3"
          />
          <text
            x={(sourceBlock.position.x + targetBlock.position.x) / 2}
            y={(sourceBlock.position.y + targetBlock.position.y) / 2 - 10}
            text-anchor="middle"
            fill="black"
          >
            {connection.type}
          </text>
          <circle
            cx={(sourceBlock.position.x + targetBlock.position.x) / 2}
            cy={(sourceBlock.position.y + targetBlock.position.y) / 2}
            r="5"
            fill="red"
            on:click={() => deleteConnection(connection.id)}
          />
        {/if}
      {/each}
      {#if isConnecting && connectionStart && connectionEnd}
        {@const startBlock = $flow.blocks.find((b) => b.id === connectionStart)}
        {#if startBlock}
          <path
            d={generatePath(startBlock.position, connectionEnd)}
            fill="none"
            stroke={connectionType === "next"
              ? "green"
              : connectionType === "alternative"
                ? "blue"
                : "red"}
            stroke-width="3"
            stroke-dasharray="5,5"
          />
        {/if}
      {/if}
    </svg>
    {#each $filteredBlocks as block (block.id)}
      <div
        class="block {block.type} {highlightedBlockId === block.id
          ? 'highlight'
          : ''}"
        style="left: {block.position.x}px; top: {block.position
          .y}px; background-color: {blockColors[block.type]};"
        draggable="true"
        on:dragstart={(e) => handleDragStart(e, block.id)}
        on:click={() => openPropertiesModal(block)}
        on:contextmenu={(e) => openContextMenu(e, block.id)}
        data-id={block.id}
        transition:fade
      >
        <h3>{blockTypes[block.type].name}</h3>
        <div class="connect-button-group">
          <button
            class="connect-button next"
            title="Next Connection"
            on:mousedown|stopPropagation={(e) => {
              selectConnectionType("next");
              startConnection(block.id, e);
            }}
          >
            <svg
              xmlns="http://www.w3.org/2000/svg"
              viewBox="0 0 24 24"
              width="16"
              height="16"
              ><path fill="none" d="M0 0h24v24H0z" /><path
                d="M13.172 12l-4.95-4.95 1.414-1.414L16 12l-6.364 6.364-1.414-1.414z"
                fill="currentColor"
              /></svg
            >
          </button>
          <button
            class="connect-button alternative"
            title="Alternative Connection"
            on:mousedown|stopPropagation={(e) => {
              selectConnectionType("alternative");
              startConnection(block.id, e);
            }}
          >
            <svg
              xmlns="http://www.w3.org/2000/svg"
              viewBox="0 0 24 24"
              width="16"
              height="16"
              ><path fill="none" d="M0 0h24v24H0z" /><path
                d="M12 3c.825 0 1.5.675 1.5 1.5S12.825 6 12 6s-1.5-.675-1.5-1.5S11.175 3 12 3zm0 15c.825 0 1.5.675 1.5 1.5S12.825 21 12 21s-1.5-.675-1.5-1.5.675-1.5 1.5-1.5zm0-7.5c.825 0 1.5.675 1.5 1.5s-.675 1.5-1.5 1.5-1.5-.675-1.5-1.5.675-1.5 1.5-1.5z"
                fill="currentColor"
              /></svg
            >
          </button>
          <button
            class="connect-button error"
            title="Error Connection"
            on:mousedown|stopPropagation={(e) => {
              selectConnectionType("error");
              startConnection(block.id, e);
            }}
          >
            <svg
              xmlns="http://www.w3.org/2000/svg"
              viewBox="0 0 24 24"
              width="16"
              height="16"
              ><path fill="none" d="M0 0h24v24H0z" /><path
                d="M12 22C6.477 22 2 17.523 2 12S6.477 2 12 2s10 4.477 10 10-4.477 10-10 10zm-1-7v2h2v-2h-2zm0-8v6h2V7h-2z"
                fill="currentColor"
              /></svg
            >
          </button>
        </div>
      </div>
    {/each}
  </div>

  <div class="minimap">
    <svg width="100%" height="100%" viewBox="0 0 1000 1000">
      <g
        class="minimap-content"
        style="transform: scale({1 / zoom}) translate({-pan.x}px, {-pan.y}px)"
      >
        {#each $flow.blocks as block}
          <rect
            x={block.position.x / 5}
            y={block.position.y / 5}
            width="30"
            height="20"
            fill={blockColors[block.type]}
          />
        {/each}
      </g>
      <rect
        x={-pan.x / 5 / zoom}
        y={-pan.y / 5 / zoom}
        width={200 / zoom}
        height={150 / zoom}
        fill="none"
        stroke="red"
        stroke-width="2"
      />
    </svg>
  </div>

  {#if showPropertiesModal && $selectedBlock}
    <div class="modal" transition:fly={{ y: 200, duration: 300 }}>
      <div class="modal-content">
        <h2>Block Properties: {blockTypes[$selectedBlock.type].name}</h2>
        {#if $selectedBlock.type === "api_call"}
          <label>
            URL:
            <input type="text" bind:value={$selectedBlock.data.url} />
          </label>
          <label>
            Method:
            <select bind:value={$selectedBlock.data.method}>
              <option value="GET">GET</option>
              <option value="POST">POST</option>
              <option value="PUT">PUT</option>
              <option value="DELETE">DELETE</option>
            </select>
          </label>
          <label>
            Headers:
            <textarea bind:value={$selectedBlock.data.headers}></textarea>
          </label>
          <label>
            Body:
            <textarea bind:value={$selectedBlock.data.body}></textarea>
          </label>
        {:else if $selectedBlock.type === "condition"}
          <label>
            Condition:
            <input type="text" bind:value={$selectedBlock.data.condition} />
          </label>
        {:else if $selectedBlock.type === "loop"}
          <label>
            Iterator:
            <input type="text" bind:value={$selectedBlock.data.iterator} />
          </label>
          <label>
            Max Iterations:
            <input
              type="number"
              bind:value={$selectedBlock.data.maxIterations}
            />
          </label>
        {:else if $selectedBlock.type === "variable"}
          <label>
            Name:
            <input type="text" bind:value={$selectedBlock.data.name} />
          </label>
          <label>
            Value:
            <input type="text" bind:value={$selectedBlock.data.value} />
          </label>
        {:else if $selectedBlock.type === "json_transformer"}
          <label>
            Transform Function:
            <textarea bind:value={$selectedBlock.data.transformFunction}
            ></textarea>
          </label>
        {/if}
        <button on:click={updateBlockProperties}>Save</button>
        <button on:click={closePropertiesModal}>Cancel</button>
      </div>
    </div>
  {/if}

  {#if showVariablesPanel}
    <div class="modal" transition:fly={{ x: 200, duration: 300 }}>
      <div class="modal-content">
        <h2>Variables</h2>
        {#each Object.entries($flow.variables) as [key, value]}
          <div>
            <input type="text" bind:value={key} />
            <input type="text" bind:value />
            <button on:click={() => deleteVariable(key)}>Delete</button>
          </div>
        {/each}
        <button on:click={addVariable}>Add Variable</button>
        <button on:click={() => (showVariablesPanel = false)}>Close</button>
      </div>
    </div>
  {/if}

  {#if showHelpPanel}
    <div class="modal" transition:fly={{ x: -200, duration: 300 }}>
      <div class="modal-content">
        <h2>Help</h2>
        <h3>Block Types</h3>
        <ul>
          {#each Object.entries(blockTypes) as [type, { name }]}
            <li>
              <strong>{name}:</strong>
              {#if type === "api_call"}
                Used to make API calls to external services.
              {:else if type === "condition"}
                Allows for conditional branching in the flow.
              {:else if type === "loop"}
                Repeats a set of actions for a specified number of times or
                until a condition is met.
              {:else if type === "variable"}
                Defines a variable that can be used throughout the flow.
              {:else if type === "json_transformer"}
                Transforms JSON data from one format to another.
              {:else if type === "custom_script"}
                Executes custom JavaScript code.
              {:else if type === "webhook"}
                Listens for incoming HTTP requests.
              {:else if type === "auth"}
                Handles authentication for API calls.
              {/if}
            </li>
          {/each}
        </ul>
        <h3>Keyboard Shortcuts</h3>
        <ul>
          <li><strong>Ctrl/Cmd + Z:</strong> Undo</li>
          <li><strong>Ctrl/Cmd + Y:</strong> Redo</li>
          <li><strong>Ctrl/Cmd + S:</strong> Save Flow</li>
          <li><strong>Ctrl/Cmd + F:</strong> Focus Search</li>
          <li><strong>Ctrl/Cmd + G:</strong> Auto Arrange Blocks</li>
        </ul>
        <button on:click={() => (showHelpPanel = false)}>Close</button>
      </div>
    </div>
  {/if}

  {#if showContextMenu}
    <div
      class="context-menu"
      style="left: {contextMenuPosition.x}px; top: {contextMenuPosition.y}px;"
    >
      <button on:click={duplicateBlock}>Duplicate</button>
      <button
        on:click={() => {
          if (contextMenuTargetId) {
            deleteBlock(contextMenuTargetId);
            closeContextMenu();
          }
        }}>Delete</button
      >
      <button
        on:click={() => {
          if (contextMenuTargetId) {
            const block = $flow.blocks.find(
              (b) => b.id === contextMenuTargetId
            );
            if (block) {
              openPropertiesModal(block);
              closeContextMenu();
            }
          }
        }}>Edit Properties</button
      >
    </div>
  {/if}
</div>
