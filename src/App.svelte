<script>
  import { onMount } from "svelte";
  import { fly } from "svelte/transition";
  import Tailwind from "./Tailwind.svelte";
  import PDFPage from "./PDFPage.svelte";
  import Image from "./Image.svelte";
  import Text from "./Text.svelte";
  import Drawing from "./Drawing.svelte";
  import DrawingCanvas from "./DrawingCanvas.svelte";
  import prepareAssets, { fetchFont } from "./utils/prepareAssets.js";
  import {
    readAsArrayBuffer,
    readAsImage,
    readAsPDF,
    readAsDataURL,
  } from "./utils/asyncReader.js";
  import { ggID } from "./utils/helper.js";
  import { save } from "./utils/PDF.js";

  const genID = ggID();
  let pdfFile;
  let pdfName = "";
  let pages = [];
  let pagesScale = [];
  let allObjects = [];
  let currentFont = "Times-Roman";
  let focusId = null;
  let selectedPageIndex = -1;
  let saving = false;
  let addingDrawing = false;

  // for test purpose
  onMount(async () => {
    try {
      const res = await fetch("/test.pdf");
      const pdfBlob = await res.blob();
      await addPDF(pdfBlob);
      selectedPageIndex = 0;
      setTimeout(() => {
        fetchFont(currentFont);
        prepareAssets();
      }, 5000);
    } catch (e) {
      console.log(e);
    }
  });

  async function onUploadPDF(pdfId) {
    try {
      const response = await fetch(
        `http://localhost:8001/pdfs?file_id=${pdfId}`,
      );
      if (!response.ok) {
        throw new Error("Failed to fetch PDF from the server");
      }

      const pdfBlob = await response.blob();
      const file = new File([pdfBlob], "pdf_from_database.pdf", {
        type: "application/pdf",
      });

      selectedPageIndex = -1;
      await addPDF(file);
      selectedPageIndex = 0;
    } catch (error) {
      console.error(error);
    }
  }

  async function addPDF(file) {
    try {
      const pdf = await readAsPDF(file);
      pdfName = file.name;
      pdfFile = file;
      const numPages = pdf.numPages;
      pages = Array(numPages)
        .fill()
        .map((_, i) => pdf.getPage(i + 1));
      allObjects = pages.map(() => []);
      pagesScale = Array(numPages).fill(1);
    } catch (e) {
      console.log("Failed to add pdf.");
      throw e;
    }
  }

  async function onUploadImage(e) {
    const file = e.target.files[0];
    if (file && selectedPageIndex >= 0) {
      addImage(file);
    }
    e.target.value = null;
  }

  async function addImage(file) {
    try {
      const url = await readAsDataURL(file);
      const img = await readAsImage(url);
      const id = genID();
      const { width, height } = img;
      const object = {
        id,
        type: "image",
        width,
        height,
        x: 0,
        y: 0,
        payload: img,
        file,
      };
      allObjects = allObjects.map((objects, pIndex) =>
        pIndex === selectedPageIndex ? [...objects, object] : objects,
      );
    } catch (e) {
      console.log(`Fail to add image.`, e);
    }
  }

  function onAddTextField() {
    if (selectedPageIndex >= 0) {
      addTextField();
    }
  }

  function addTextField(text = "New Text Field") {
    const id = genID();
    fetchFont(currentFont);
    const object = {
      id,
      text,
      type: "text",
      size: 16,
      width: 0,
      lineHeight: 1.4,
      fontFamily: currentFont,
      x: 0,
      y: 0,
    };
    allObjects = allObjects.map((objects, pIndex) =>
      pIndex === selectedPageIndex ? [...objects, object] : objects,
    );
  }

  function onAddDrawing() {
    if (selectedPageIndex >= 0) {
      addingDrawing = true;
    }
  }

  function addDrawing(originWidth, originHeight, path, scale = 1) {
    const id = genID();
    const object = {
      id,
      path,
      type: "drawing",
      x: 0,
      y: 0,
      originWidth,
      originHeight,
      width: originWidth * scale,
      scale,
    };
    allObjects = allObjects.map((objects, pIndex) =>
      pIndex === selectedPageIndex ? [...objects, object] : objects,
    );
  }

  function selectFontFamily(event) {
    const name = event.detail.name;
    fetchFont(name);
    currentFont = name;
  }

  function selectPage(index) {
    selectedPageIndex = index;
  }

  function updateObject(objectId, payload) {
    allObjects = allObjects.map((objects, pIndex) =>
      pIndex == selectedPageIndex
        ? objects.map((object) =>
            object.id === objectId ? { ...object, ...payload } : object,
          )
        : objects,
    );
  }

  function deleteObject(objectId) {
    allObjects = allObjects.map((objects, pIndex) =>
      pIndex == selectedPageIndex
        ? objects.filter((object) => object.id !== objectId)
        : objects,
    );
  }

  function onMeasure(scale, i) {
    pagesScale[i] = scale;
  }

  async function updatePDF(pdfId, userEmail, savedPdfData) {
    const formData = new FormData();
    formData.append("file_id", pdfId);
    formData.append("user_email", userEmail);
    formData.append("pdf", savedPdfData);

    try {
      const response = await fetch(`http://localhost:8001/pdfs/sign`, {
        method: "POST",
        body: formData,
      });

      return response;
    } catch (e) {
      throw new Error(`Error updating PDF: ${e}`);
    }
  }

  async function savePDF() {
    const pdfId = "66055a0ff430a5cd90047b58";
    const userEmail = "user2@example.com";

    if (!pdfFile || saving || !pages.length) return;
    saving = true;

    try {
      const savedPdfData = await save(pdfFile, allObjects, pdfName, pagesScale);
      const response = await updatePDF(pdfId, userEmail, savedPdfData);
      if (!response.ok) {
        throw new Error(`Error updating PDF: ${await response.text()}`);
      }

      console.log("PDF saved and updated successfully!");
    } catch (e) {
      console.error("Error saving or updating PDF:", e);
    } finally {
      saving = false;
    }
  }
</script>

<svelte:window
  on:dragenter|preventDefault
  on:dragover|preventDefault
  on:drop|preventDefault={onUploadPDF("66055a0ff430a5cd90047b58")}
/>

<Tailwind />

<main class="flex flex-col items-center py-16 bg-gray-100 min-h-screen">
  <div
    class="fixed z-10 top-0 left-0 right-0 h-12 flex justify-center items-center bg-gray-200 border-b border-gray-300"
  >
    <input
      type="file"
      id="image"
      name="image"
      class="hidden"
      on:change={onUploadImage}
    />
    <div
      class="relative mr-3 flex h-8 bg-gray-400 rounded-sm overflow-hidden md:mr-4"
    >
      <label
        for="image"
        class="flex items-center justify-center h-full w-8 hover:bg-gray-500 cursor-pointer"
        class:cursor-not-allowed={selectedPageIndex < 0}
        class:bg-gray-500={selectedPageIndex < 0}
      >
        <img src="image.svg" alt="An icon for adding images" />
      </label>
      <label
        for="text"
        class="flex items-center justify-center h-full w-8 hover:bg-gray-500 cursor-pointer"
        class:cursor-not-allowed={selectedPageIndex < 0}
        class:bg-gray-500={selectedPageIndex < 0}
        on:click={onAddTextField}
      >
        <img src="notes.svg" alt="An icon for adding text" />
      </label>
      <label
        on:click={onAddDrawing}
        class="flex items-center justify-center h-full w-8 hover:bg-gray-500 cursor-pointer"
        class:cursor-not-allowed={selectedPageIndex < 0}
        class:bg-gray-500={selectedPageIndex < 0}
      >
        <img src="gesture.svg" alt="An icon for adding drawing" />
      </label>
    </div>
    <div class="justify-center mr-3 md:mr-4 w-full max-w-xs hidden md:flex">
      <img src="/edit.svg" class="mr-2" alt="a pen, edit pdf name" />
      <input
        placeholder="Rename your PDF here"
        type="text"
        class="flex-grow bg-transparent"
        bind:value={pdfName}
      />
    </div>
    <button
      on:click={savePDF}
      class="w-20 bg-blue-500 hover:bg-blue-700 text-white font-bold py-1 px-3 md:px-4 mr-3 md:mr-4 rounded"
      class:cursor-not-allowed={pages.length === 0 || saving || !pdfFile}
      class:bg-blue-700={pages.length === 0 || saving || !pdfFile}
    >
      {saving ? "Saving" : "Save"}
    </button>
    <a href="https://github.com/ShizukuIchi/pdf-editor">
      <img
        src="/GitHub-Mark-32px.png"
        alt="A GitHub icon leads to personal GitHub page"
      />
    </a>
  </div>
  {#if addingDrawing}
    <div
      transition:fly={{ y: -200, duration: 500 }}
      class="fixed z-10 top-0 left-0 right-0 border-b border-gray-300 bg-white shadow-lg"
      style="height: 50%;"
    >
      <DrawingCanvas
        on:finish={(e) => {
          const { originWidth, originHeight, path } = e.detail;
          let scale = 1;
          if (originWidth > 500) {
            scale = 500 / originWidth;
          }
          addDrawing(originWidth, originHeight, path, scale);
          addingDrawing = false;
        }}
        on:cancel={() => (addingDrawing = false)}
      />
    </div>
  {/if}
  {#if pages.length}
    <div class="flex justify-center px-5 w-full md:hidden">
      <img src="/edit.svg" class="mr-2" alt="a pen, edit pdf name" />
      <input
        placeholder="Rename your PDF here"
        type="text"
        class="flex-grow bg-transparent"
        bind:value={pdfName}
      />
    </div>
    <div class="w-full">
      {#each pages as page, pIndex (page)}
        <div
          class="p-5 w-full flex flex-col items-center overflow-hidden"
          on:mousedown={() => selectPage(pIndex)}
          on:touchstart={() => selectPage(pIndex)}
        >
          <div
            class="relative shadow-lg"
            class:shadow-outline={pIndex === selectedPageIndex}
          >
            <PDFPage
              on:measure={(e) => onMeasure(e.detail.scale, pIndex)}
              {page}
            />
            <div
              class="absolute top-0 left-0 transform origin-top-left"
              style="transform: scale({pagesScale[
                pIndex
              ]}); touch-action: none;"
            >
              {#each allObjects[pIndex] as object (object.id)}
                {#if object.type === "image"}
                  <Image
                    on:update={(e) => updateObject(object.id, e.detail)}
                    on:delete={() => deleteObject(object.id)}
                    file={object.file}
                    payload={object.payload}
                    x={object.x}
                    y={object.y}
                    width={object.width}
                    height={object.height}
                    pageScale={pagesScale[pIndex]}
                  />
                {:else if object.type === "text"}
                  <Text
                    on:update={(e) => updateObject(object.id, e.detail)}
                    on:delete={() => deleteObject(object.id)}
                    on:selectFont={selectFontFamily}
                    text={object.text}
                    x={object.x}
                    y={object.y}
                    size={object.size}
                    lineHeight={object.lineHeight}
                    fontFamily={object.fontFamily}
                    pageScale={pagesScale[pIndex]}
                  />
                {:else if object.type === "drawing"}
                  <Drawing
                    on:update={(e) => updateObject(object.id, e.detail)}
                    on:delete={() => deleteObject(object.id)}
                    path={object.path}
                    x={object.x}
                    y={object.y}
                    width={object.width}
                    originWidth={object.originWidth}
                    originHeight={object.originHeight}
                    pageScale={pagesScale[pIndex]}
                  />
                {/if}
              {/each}
            </div>
          </div>
        </div>
      {/each}
    </div>
  {:else}
    <div class="w-full flex-grow flex justify-center items-center">
      <span class=" font-bold text-3xl text-gray-500">Drag something here</span>
    </div>
  {/if}
</main>
