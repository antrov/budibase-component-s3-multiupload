<script>
  import { getContext } from "svelte";

  export let bucket;
  export let onFileDrop;
  export let datasourceId;

  const handleFileDrop = (e, uuid) => {
    if (onFileDrop) {
      console.log("File drop started to process");
      onFileDrop({ file: e, uuid: uuid });
      console.log("File drop processed");
    }
  };

  const { API, notificationStore, styleable, uploadStore } = getContext("sdk");
  const component = getContext("component");

  // onMount(() => {
  //   uploadStore.actions.registerFileUpload($component.id, uploadByAPI)
  // })

  // onDestroy(() => {
  //   uploadStore.actions.unregisterFileUpload($component.id)
  // })

  let drop_zone;

  let status = "";

  function handleDragEnter(e) {
    status = "You are dragging over the " + e.target.getAttribute("id");
  }

  function handleDragLeave(e) {
    status = "You left the " + e.target.getAttribute("id");
  }

  function handleDragDrop(ev) {
    ev.preventDefault();
    const files = ev.dataTransfer.items
      ? [...ev.dataTransfer.items]
          .filter((item) => item.kind === "file")
          .map((item) => item.getAsFile())
      : [...ev.dataTransfer.files];
    processFiles(files);
  }

  function handleFileInput(event) {
    const files = [...event.target.files];
    processFiles(files);
  }

  function uuidv4() {
    return "10000000-1000-4000-8000-100000000000".replace(/[018]/g, (c) =>
      (
        +c ^
        (crypto.getRandomValues(new Uint8Array(1))[0] & (15 >> (+c / 4)))
      ).toString(16),
    );
  }

  async function processFiles(files) {
    for (const file of files) {
      console.log(`Processing file: ${file.name}`);
      const uuid = uuidv4();

      // await uploadFile(file);
      await uploadByAPI(file)
      // await uploadByMinio(file);
      handleFileDrop(file, uuid);
    }
  }

  async function uploadByAPI(data) {
    try {
      const res = await API.externalUpload({
        datasourceId: datasourceId,
        bucket: bucket,
        key: "test.txt",
        data: data,
      });
      notificationStore.actions.success("File uploaded successfully");
      return res;
    } catch (error) {
      notificationStore.actions.error(
        `Error uploading file to ${datasourceId} for ${data.name}: ${error?.message || error}`,
      );
    }
  }

  async function uploadFile(file) {
    try {
      const formData = new FormData();
      formData.append("file", file);

      const response = await fetch("/upload", {
        method: "POST",
        body: formData,
      });

      if (response.ok) {
        console.log(`File ${file.name} uploaded successfully.`);
      } else {
        console.error(`Failed to upload file ${file.name}.`);
      }
    } catch (error) {
      console.error(`Error uploading file ${file.name}:`, error);
    }
  }
</script>

<div use:styleable={$component.styles}>
  <h2 id="app_status">Drag status: {status}</h2>
  <h1>Drop Zone</h1>

  <div
    role="button"
    on:dragenter={handleDragEnter}
    on:dragleave={handleDragLeave}
    on:drop={handleDragDrop}
    on:dragover={(e) => e.preventDefault()}
    on:click={() => {
      document.getElementById("input_file").click();
    }}
    on:keydown={(e) => {
      if (e.key === "Enter" || e.key === " ") {
        document.getElementById("input_file").click();
      }
    }}
    bind:this={drop_zone}
    id="drop_zone"
    tabindex="0"
  >
    This is a custom component. The bucket setting is: {bucket} for {datasourceId}.<br
    />
    The component name is: {$component.name}.
    <input
      type="file"
      id="input_file"
      hidden
      multiple
      on:change={handleFileInput}
    />
  </div>
</div>

<style>
  #drop_zone {
    border: 2px dashed #000;
    padding: 20px;
    margin: 20px;
    transition:
      background-color 0.3s,
      border-color 0.3s;
  }

  #drop_zone:hover {
    background-color: #f0f8ff;
    border-color: #007bff;
  }
</style>
