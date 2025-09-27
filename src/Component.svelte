<script>
  import { getContext } from "svelte";

  export let bucket;
  export let prefix;
  export let onFilesDropped;
  export let onUploadStarted;
  export let onUploadSucceded;
  export let onUploadFailed;
  export let datasourceId;

  const { API, styleable, Provider } = getContext("sdk");
  const component = getContext("component");

  let filesInQueue = 0;
  let filesUploaded = 0;
  let hasHover = false;

  $: dataContext = {
    filesInQueue,
    filesUploaded,
    hasHover,
  };

  const invokeFilesDropped = (files) => {
    console.log(`Uploading ${files.length} files`);
    if (onFilesDropped) {
      onFilesDropped({ files });
    }
  };

  const invokeUploadStarted = (index, filename, uuid) => {
    console.log(
      `Upload started of ${index} file with name ${filename} and uuid ${uuid}`,
    );
    if (onUploadStarted) {
      onUploadStarted({ filename, uuid, key });
    }
  };

  const invokeUploadSucceded = (index, filename, uuid) => {
    console.log(
      `Completed of ${index} file with name ${filename} and uuid ${uuid}`,
    );
    if (onUploadSucceded) {
      onUploadSucceded({ filename, uuid });
    }
  };

  const invokeUploadFailed = (index, filename, uuid, error) => {
    console.log(
      `Failed of ${index} file with name ${filename} and uuid ${uuid}: ${error}`,
    );
    if (onUploadFailed) {
      onUploadFailed({ filename, uuid, error });
    }
  };

  let drop_zone;

  function handleDragEnter(_) {
    hasHover = true;
  }

  function handleDragLeave(_) {
    hasHover = false;
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

  function processFiles(files) {
    const uuids = files.map((file) => {
      const uuid = uuidv4();
      return {
        uuid: uuid,
        file: file,
        filename: file.name,
        key: prefix ? `${prefix}${uuid}` : uuid,
      };
    });
    invokeFilesDropped(
      uuids.map(({ filename, uuid, key }) => ({ filename, uuid, key })),
    );
    uploadUuidedFiles(uuids);
  }

  function uploadUuidedFiles(uuids) {
    initUploadCounter(uuids.length);
    uuids.reduce((p, { uuid, file, key }, index) => {
      return p
        .then(() => {
          invokeUploadStarted(index, file.name, uuid);
        })
        .then(() => {
          return uploadByAPI(index, key, file);
        })
        .then(() => {
          increaseUploadCounter();
          invokeUploadSucceded(index, file.name, uuid);
        })
        .catch((error) => {
          invokeUploadFailed(index, file.name, uuid, error);
        });
    }, Promise.resolve());
  }

  function initUploadCounter(queueSize) {
    filesInQueue += queueSize;
  }

  function increaseUploadCounter() {
    filesUploaded++;
    if (filesUploaded === filesInQueue) {
      filesInQueue = 0;
      filesUploaded = 0;
    }
  }

  async function uploadByAPI(index, key, file) {
    console.log(
      `Start upload of ${index} file with name ${file.name} and key ${key}`,
    );
    return await API.externalUpload({
      datasourceId: datasourceId,
      bucket: bucket,
      key: key,
      data: file,
    });
  }
</script>

<div use:styleable={$component.styles} class="drag-and-drop-zone">
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
    <Provider data={dataContext}>
      <slot />
    </Provider>
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
  .drag-and-drop-zone {
    cursor: pointer;
    transition:
      background-color 0.3s,
      border-color 0.3s;
  }

  .drag-and-drop-zone:hover {
    background-color: #f0f8ff;
    border-color: #007bff;
  }
</style>
