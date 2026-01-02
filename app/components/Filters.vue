<script setup lang="ts">
import {
  getAllTags,
  getSearchSuggestions,
  SearchSuggestionType,
  type ApiHttpError,
  type TagResponseDto,
} from "@immich/sdk";

const tagIds = defineModel<string[]>("tagIds", { default: [] });
const cameraModel = defineModel<string | null>("cameraModel", {
  default: null,
});
const isNotInAlbum = defineModel<boolean>("isNotInAlbum", { default: false });
const pageSize = defineModel<number>("pageSize", { default: 10 });

const {
  data: tags,
  status: tagsStatus,
  error: tagsError,
} = useAsyncData<TagResponseDto[], ApiHttpError>(() => getAllTags());
const {
  data: cameraModels,
  status: cameraStatus,
  error: cameraError,
} = useAsyncData<
  string[],
  ApiHttpError,
  Array<{ value: string; label: string }>
>(() => getSearchSuggestions({ $type: SearchSuggestionType.CameraModel }), {
  transform: (models) =>
    models.map((model) => ({
      value: model,
      label: model || "Unknown",
    })),
});

const pageOptions = [5, 10, 20, 50].map((count) => ({
  label: count.toString(),
  value: count,
}));
</script>

<template>
  <div class="flex gap-6">
    <div>
      <MultiSelect
        v-model="tagIds"
        :max-selected-labels="3"
        :options="tags"
        option-label="name"
        option-value="id"
        placeholder="Choose tags"
        :loading="tagsStatus === 'pending'"
        show-clear
        filter
        :pt="{overlay: {style: 'z-index: 10001'}}"
      />
      <Message v-if="tagsError" severity="error" size="small" variant="simple">
        Error {{ tagsError.data?.statusCode ?? tagsError.status }}:
        {{ tagsError.data?.message ?? tagsError.message }}
      </Message>
    </div>
    <div>
      <Select
        v-model="cameraModel"
        :options="cameraModels"
        option-label="label"
        option-value="value"
        placeholder="Camera model"
        :loading="cameraStatus === 'pending'"
        show-clear
        filter
        :pt="{overlay: {style: 'z-index: 10001'}}"
      />
      <Message
        v-if="cameraError"
        severity="error"
        size="small"
        variant="simple"
      >
        Error {{ cameraError.data?.statusCode ?? cameraError.status }}:
        {{ cameraError.data?.message ?? cameraError.message }}
      </Message>
    </div>
    <div class="flex items-center gap-2">
      <Checkbox v-model="isNotInAlbum" inputId="not-in-album" binary />
      <label for="not-in-album">Not in any albums</label>
    </div>
    <div>
      <Select
        v-model="pageSize"
        :options="pageOptions"
        placeholder="Items per page"
        option-label="label"
        option-value="value"
        filter
        :pt="{overlay: {style: 'z-index: 10001'}}"
      />
    </div>
  </div>
</template>
