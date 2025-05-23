<script setup lang="ts">
import type { FileUploadSelectEvent } from "primevue/fileupload";
import { useTimeline } from "~/lib/timeline";

const isServerInited = ref(false);

// Location history export timeline

const { setTimeline } = useTimeline();

async function uploadTimeline(event: FileUploadSelectEvent) {
  const file: File = event.files[0];
  setTimeline(JSON.parse(await file.text()));
}

// Filters

const cameraModel = useLocalStorage<string | null>("camera-model", null);
const tagIds = useLocalStorage<string[]>("tag-ids", []);
const isNotInAlbum = useLocalStorage("is-not-in-album", false);
const pageSize = useLocalStorage("page-size", 10);
</script>

<template>
  <div>
    <header class="grid gap-2">
      <Panel toggleable collapsed header="README">
        <Message>
          In the location history, <code>timelinePath</code> segments are given
          priority due to their precision, followed by
          <code>activity</code> segments, and lastly
          <code>visit</code> segments.
          <br />
          After
          <a href="https://github.com/immich-app/immich/pull/17061">
            <code>immich#17061</code>
          </a>
          is merged, it will be possible to use visually similar images to
          estimate the location of the image.
        </Message>
      </Panel>
      <ServerConfig
        @init="
          ({ apiKey, baseUrl }) =>
            (isServerInited = Boolean(apiKey) && Boolean(baseUrl))
        "
      >
        <template #extra>
          <FileUpload
            mode="basic"
            custom-upload
            accept="application/json"
            choose-label="Upload Timeline.json"
            @select="uploadTimeline"
          />
        </template>
      </ServerConfig>
      <Panel v-if="isServerInited" header="Configuration" toggleable>
        <div>
          <h2 class="text-lg font-semibold">Image search options</h2>
          <Filters
            v-model:cameraModel="cameraModel"
            v-model:tagIds="tagIds"
            v-model:isNotInAlbum="isNotInAlbum"
            v-model:page-size="pageSize"
          />
        </div>
      </Panel>
    </header>
  </div>
  <main v-if="isServerInited" class="mt-4">
    <ImagesSearch :tag-ids :is-not-in-album :camera-model :page-size />
  </main>
</template>
