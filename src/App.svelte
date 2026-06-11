<script lang="ts">
  import { onDestroy } from 'svelte';
  import { PDFDocument } from 'pdf-lib';

  type PageMode = 'a4-portrait' | 'a4-landscape' | 'original';
  type MarginMode = 'none' | 'standard';

  type ImageItem = {
    id: string;
    file: File;
    name: string;
    size: number;
    previewUrl: string;
    width: number;
    height: number;
  };

  const A4_PORTRAIT = { width: 595.28, height: 841.89 };
  const A4_LANDSCAPE = { width: 841.89, height: 595.28 };
  const STANDARD_MARGIN = 28.35;
  const SUPPORTED_TYPES = new Set(['image/jpeg', 'image/png', 'image/webp']);

  let items: ImageItem[] = [];
  let pageMode: PageMode = 'a4-portrait';
  let marginMode: MarginMode = 'standard';
  let quality = 80;
  let isUploadDragging = false;
  let isGenerating = false;
  let draggedIndex: number | null = null;
  let errorMessage = '';
  let fileInput: HTMLInputElement;

  $: totalInputSize = items.reduce((sum, item) => sum + item.size, 0);
  $: canGenerate = items.length > 0 && !isGenerating;

  onDestroy(() => {
    items.forEach((item) => URL.revokeObjectURL(item.previewUrl));
  });

  function formatBytes(bytes: number) {
    if (bytes === 0) return '0 B';
    const units = ['B', 'KB', 'MB', 'GB'];
    const exponent = Math.min(Math.floor(Math.log(bytes) / Math.log(1024)), units.length - 1);
    const value = bytes / 1024 ** exponent;
    return `${value.toFixed(value >= 10 || exponent === 0 ? 0 : 1)} ${units[exponent]}`;
  }

  function getPageSize(item: ImageItem) {
    if (pageMode === 'a4-landscape') return A4_LANDSCAPE;
    if (pageMode === 'original') return { width: item.width, height: item.height };
    return A4_PORTRAIT;
  }

  function getPageLabel() {
    if (pageMode === 'a4-landscape') return 'A4 横';
    if (pageMode === 'original') return '画像サイズ';
    return 'A4 縦';
  }

  function getMargin() {
    return marginMode === 'standard' ? STANDARD_MARGIN : 0;
  }

  function readImageDimensions(url: string) {
    return new Promise<{ width: number; height: number }>((resolve, reject) => {
      const image = new Image();
      image.onload = () => resolve({ width: image.naturalWidth, height: image.naturalHeight });
      image.onerror = () => reject(new Error('画像を読み込めませんでした。'));
      image.src = url;
    });
  }

  async function addFiles(fileList: FileList | File[]) {
    errorMessage = '';
    const files = Array.from(fileList);
    const imageFiles = files.filter((file) => SUPPORTED_TYPES.has(file.type));

    if (files.length > 0 && imageFiles.length === 0) {
      errorMessage = 'JPG、PNG、WebPの画像を選択してください。';
      return;
    }

    const loadedItems: ImageItem[] = [];

    for (const file of imageFiles) {
      const previewUrl = URL.createObjectURL(file);

      try {
        const dimensions = await readImageDimensions(previewUrl);
        loadedItems.push({
          id: `${file.name}-${file.lastModified}-${crypto.randomUUID()}`,
          file,
          name: file.name,
          size: file.size,
          previewUrl,
          width: dimensions.width,
          height: dimensions.height
        });
      } catch (error) {
        URL.revokeObjectURL(previewUrl);
        errorMessage = error instanceof Error ? error.message : '画像を読み込めませんでした。';
      }
    }

    items = [...items, ...loadedItems];

    if (files.length !== imageFiles.length) {
      errorMessage = '対応していないファイルはスキップしました。';
    }
  }

  function handleFileInput(event: Event) {
    const input = event.currentTarget as HTMLInputElement;
    if (input.files) void addFiles(input.files);
    input.value = '';
  }

  function openFilePicker() {
    fileInput?.click();
  }

  function handleUploadDragOver(event: DragEvent) {
    event.preventDefault();
    isUploadDragging = true;
  }

  function handleUploadDragLeave(event: DragEvent) {
    if (event.currentTarget === event.target) {
      isUploadDragging = false;
    }
  }

  function handleDrop(event: DragEvent) {
    event.preventDefault();
    isUploadDragging = false;

    if (event.dataTransfer?.files) {
      void addFiles(event.dataTransfer.files);
    }
  }

  function removeItem(id: string) {
    const item = items.find((candidate) => candidate.id === id);
    if (item) URL.revokeObjectURL(item.previewUrl);
    items = items.filter((candidate) => candidate.id !== id);
  }

  function clearItems() {
    items.forEach((item) => URL.revokeObjectURL(item.previewUrl));
    items = [];
    errorMessage = '';
  }

  function moveItem(fromIndex: number, toIndex: number) {
    if (fromIndex === toIndex || fromIndex < 0 || toIndex < 0) return;
    const nextItems = [...items];
    const [movedItem] = nextItems.splice(fromIndex, 1);
    nextItems.splice(toIndex, 0, movedItem);
    items = nextItems;
  }

  function moveWithButton(index: number, direction: -1 | 1) {
    moveItem(index, index + direction);
  }

  function handleSortDragStart(index: number) {
    draggedIndex = index;
  }

  function handleSortDragOver(event: DragEvent) {
    event.preventDefault();
  }

  function handleSortDrop(index: number) {
    if (draggedIndex !== null) {
      moveItem(draggedIndex, index);
    }
    draggedIndex = null;
  }

  function handleSortDragEnd() {
    draggedIndex = null;
  }

  function loadImage(url: string) {
    return new Promise<HTMLImageElement>((resolve, reject) => {
      const image = new Image();
      image.onload = () => resolve(image);
      image.onerror = () => reject(new Error('画像を変換できませんでした。'));
      image.src = url;
    });
  }

  async function encodeAsJpeg(item: ImageItem) {
    const image = await loadImage(item.previewUrl);
    const canvas = document.createElement('canvas');
    canvas.width = item.width;
    canvas.height = item.height;

    const context = canvas.getContext('2d');
    if (!context) throw new Error('Canvasを初期化できませんでした。');

    context.fillStyle = '#ffffff';
    context.fillRect(0, 0, canvas.width, canvas.height);
    context.drawImage(image, 0, 0, canvas.width, canvas.height);

    const blob = await new Promise<Blob>((resolve, reject) => {
      canvas.toBlob(
        (result) => {
          if (result) resolve(result);
          else reject(new Error('画像の圧縮に失敗しました。'));
        },
        'image/jpeg',
        Math.max(0.01, Math.min(1, quality / 100))
      );
    });

    return new Uint8Array(await blob.arrayBuffer());
  }

  async function generatePdf() {
    if (!canGenerate) return;

    isGenerating = true;
    errorMessage = '';

    try {
      const pdfDocument = await PDFDocument.create();
      const margin = getMargin();

      for (const item of items) {
        const pageSize = getPageSize(item);
        const page = pdfDocument.addPage([pageSize.width, pageSize.height]);
        const jpegBytes = await encodeAsJpeg(item);
        const embeddedImage = await pdfDocument.embedJpg(jpegBytes);

        const availableWidth = Math.max(1, pageSize.width - margin * 2);
        const availableHeight = Math.max(1, pageSize.height - margin * 2);
        const scale = Math.min(availableWidth / item.width, availableHeight / item.height);
        const drawWidth = item.width * scale;
        const drawHeight = item.height * scale;
        const x = (pageSize.width - drawWidth) / 2;
        const y = (pageSize.height - drawHeight) / 2;

        page.drawImage(embeddedImage, {
          x,
          y,
          width: drawWidth,
          height: drawHeight
        });
      }

      const pdfBytes = await pdfDocument.save();
      const pdfArrayBuffer = new ArrayBuffer(pdfBytes.byteLength);
      new Uint8Array(pdfArrayBuffer).set(pdfBytes);
      const blob = new Blob([pdfArrayBuffer], { type: 'application/pdf' });
      const url = URL.createObjectURL(blob);
      const link = document.createElement('a');
      link.href = url;
      link.download = `img2pdf-${new Date().toISOString().slice(0, 10)}.pdf`;
      document.body.appendChild(link);
      link.click();
      link.remove();
      URL.revokeObjectURL(url);
    } catch (error) {
      errorMessage = error instanceof Error ? error.message : 'PDFの生成に失敗しました。';
    } finally {
      isGenerating = false;
    }
  }
</script>

<svelte:head>
  <title>img2pdf - 画像をPDF化</title>
</svelte:head>

<main class="app-shell">
  <section class="workspace" aria-label="画像PDF変換ツール">
    <header class="topbar">
      <div>
        <p class="eyebrow">Client-side image converter</p>
        <h1>img2pdf</h1>
      </div>
      <div class="privacy-badge">ブラウザ内で変換</div>
    </header>

    <div class="tool-layout">
      <section class="left-pane" aria-label="画像の選択と並び替え">
        <div
          class:dragging={isUploadDragging}
          class="drop-zone"
          role="button"
          tabindex="0"
          ondragover={handleUploadDragOver}
          ondragleave={handleUploadDragLeave}
          ondrop={handleDrop}
          onclick={openFilePicker}
          onkeydown={(event) => {
            if (event.key === 'Enter' || event.key === ' ') openFilePicker();
          }}
        >
          <input
            bind:this={fileInput}
            accept="image/jpeg,image/png,image/webp"
            multiple
            onchange={handleFileInput}
            type="file"
          />
          <div class="drop-icon" aria-hidden="true">+</div>
          <div>
            <h2>画像を追加</h2>
            <p>JPG / PNG / WebP を選択、またはここへドロップ</p>
          </div>
        </div>

        {#if errorMessage}
          <p class="message" role="alert">{errorMessage}</p>
        {/if}

        <div class="list-header">
          <div>
            <h2>画像リスト</h2>
            <p>{items.length}枚 / {formatBytes(totalInputSize)}</p>
          </div>
          {#if items.length > 0}
            <button class="ghost-button" type="button" onclick={clearItems}>すべて削除</button>
          {/if}
        </div>

        {#if items.length === 0}
          <div class="empty-state">
            <p>画像を追加すると、ここで順番を並び替えられます。</p>
          </div>
        {:else}
          <ol class="image-list" aria-label="PDFに入れる画像の順番">
            {#each items as item, index (item.id)}
              <li
                class:sorting={draggedIndex === index}
                class="image-row"
                draggable="true"
                ondragstart={() => handleSortDragStart(index)}
                ondragover={handleSortDragOver}
                ondrop={() => handleSortDrop(index)}
                ondragend={handleSortDragEnd}
              >
                <div class="drag-handle" aria-hidden="true">⋮⋮</div>
                <img src={item.previewUrl} alt={`${item.name} のプレビュー`} />
                <div class="image-meta">
                  <strong>{index + 1}. {item.name}</strong>
                  <span>{item.width} × {item.height}px / {formatBytes(item.size)}</span>
                </div>
                <div class="row-actions" aria-label={`${item.name} の操作`}>
                  <button
                    aria-label="1つ上へ移動"
                    class="icon-button"
                    disabled={index === 0}
                    type="button"
                    onclick={() => moveWithButton(index, -1)}
                  >
                    ↑
                  </button>
                  <button
                    aria-label="1つ下へ移動"
                    class="icon-button"
                    disabled={index === items.length - 1}
                    type="button"
                    onclick={() => moveWithButton(index, 1)}
                  >
                    ↓
                  </button>
                  <button
                    aria-label="削除"
                    class="icon-button danger"
                    type="button"
                    onclick={() => removeItem(item.id)}
                  >
                    ×
                  </button>
                </div>
              </li>
            {/each}
          </ol>
        {/if}
      </section>

      <aside class="settings-panel" aria-label="PDF設定">
        <div class="panel-section">
          <h2>出力設定</h2>
          <p>{getPageLabel()} / 画質 {quality}% / 余白{marginMode === 'standard' ? 'あり' : 'なし'}</p>
        </div>

        <fieldset>
          <legend>ページサイズ</legend>
          <label>
            <input bind:group={pageMode} type="radio" value="a4-portrait" />
            A4 縦
          </label>
          <label>
            <input bind:group={pageMode} type="radio" value="a4-landscape" />
            A4 横
          </label>
          <label>
            <input bind:group={pageMode} type="radio" value="original" />
            画像サイズそのまま
          </label>
        </fieldset>

        <fieldset>
          <legend>余白</legend>
          <label>
            <input bind:group={marginMode} type="radio" value="standard" />
            あり
          </label>
          <label>
            <input bind:group={marginMode} type="radio" value="none" />
            なし
          </label>
        </fieldset>

        <label class="quality-control">
          <span>画質</span>
          <strong>{quality}%</strong>
          <input bind:value={quality} max="100" min="1" type="range" />
        </label>

        <button class="primary-button" disabled={!canGenerate} type="button" onclick={generatePdf}>
          {isGenerating ? 'PDF生成中...' : 'PDFをダウンロード'}
        </button>

        <dl class="summary-list">
          <div>
            <dt>ページ数</dt>
            <dd>{items.length}</dd>
          </div>
          <div>
            <dt>入力サイズ</dt>
            <dd>{formatBytes(totalInputSize)}</dd>
          </div>
          <div>
            <dt>処理場所</dt>
            <dd>ローカルブラウザ</dd>
          </div>
        </dl>
      </aside>
    </div>
  </section>
</main>
