<script setup lang="ts">
import {
  searchAssets,
  updateAsset,
  searchAssetStatistics,
  type AssetResponseDto,
  type StatisticsSearchDto,
  type MetadataSearchDto,
} from "@immich/sdk";
import { segmentToGeometry, type Geometry } from "~/lib/geometry";
import { useTimeline, type LatLng } from "~/lib/timeline";

const {
  tagIds,
  isNotInAlbum,
  cameraModel,
  pageSize = 10,
  skipUnestimated = false,
} = defineProps<{
  tagIds: string[];
  isNotInAlbum: boolean;
  cameraModel: string | null;
  pageSize?: number;
  skipUnestimated?: boolean;
}>();

const { estimateLocationAtTime, timeline } = useTimeline();

type TransformedSearchResponse = {
  assets: AssetResponseDto[];
  hasNextPage: boolean;
  total: number;
};

const page = ref(1); // Note: cannot decrease page without clearing results first. Only increment.
// var instead of const to access "result" inside the function body
var {
  data: result,
  status,
  error,
  clear,
  execute,
} = useAsyncData(
  (async () => {
    const previousAssets = result?.value?.assets ?? [];

    const searchFilters = {
      country: "",
      tagIds,
      isNotInAlbum,
      model: cameraModel,
    } satisfies StatisticsSearchDto & MetadataSearchDto;

    const [{ total }, { assets }] = await Promise.all([
      searchAssetStatistics({ statisticsSearchDto: searchFilters }),
      searchAssets({
        metadataSearchDto: {
          withExif: true,
          size: pageSize,
          page: page.value,
          ...searchFilters,
        },
      }),
    ]);

    return {
      assets: [...previousAssets, ...assets.items],
      hasNextPage: assets.nextPage != null,
      total,
    };
  }) as () => Promise<TransformedSearchResponse>,
  {
    watch: [page],
  }
);

watch(
  [() => tagIds, () => isNotInAlbum, () => cameraModel, () => pageSize],
  () => {
    // Reset page to 1 if any of the search parameters (other than page) changed
    clear();
    page.value = 1;
    execute();
  }
);

type Item = {
  asset: AssetResponseDto;
  estimatedLocation: LatLng | undefined;
  geometry: Geometry[];
  confirmEdit: boolean;
};

const alreadySeen = useLocalStorage<Set<string>>("already-seen", new Set());

const allAssets = useArrayMap<AssetResponseDto, Item>(
  () => result.value?.assets ?? [],
  (asset) => {
    const {
      bestEstimate = null,
      estimateSource,
      segments = [],
    } = estimateLocationAtTime(new Date(asset.fileCreatedAt)) ?? {};

    return {
      asset,
      estimatedLocation: bestEstimate?.point,
      geometry: segments
        .map(segmentToGeometry)
        .filter((geometry) => geometry != null),
      confirmEdit:
        bestEstimate != null &&
        estimateSource === "timelinePath" &&
        !asset.originalFileName.toLowerCase().includes("screenshot"),
    };
  }
);

const items = useArrayFilter(
  allAssets,
  ({ asset }) => !alreadySeen.value.has(asset.id)
);
const alreadySeenCount = computed(
  () => allAssets.value.length - items.value.length
);

const updates = ref<Record<string, Item>>({});

// TODO: find a better way to do this
watch([items, () => skipUnestimated], ([items]) => {
  const newUpdates: Record<string, Item> = {};
  for (const item of items) {
    if (skipUnestimated && timeline != null && item.estimatedLocation == null) {
      continue;
    }
    const existing = updates.value[item.asset.id];
    newUpdates[item.asset.id] =
      existing != null && existing.estimatedLocation != null
        ? existing
        : item;
  }
  updates.value = newUpdates;
});

const confirmedUpdates = computed(() =>
  Object.values(updates.value).filter((item) => item.confirmEdit)
);

// Bulk edit mode and selection state
const bulkEditMode = ref(false);
const bulkCheckedIds = ref(new Set<string>()); // TODO: Maybe it's better to add another field like "confirmEdit" to Item instead?
const bulkLocation = ref<LatLng | undefined>(undefined);
const isBulkDialogOpen = ref(false);

const allCheckedForBulkUpdate = computed(
  () => Object.keys(updates.value).length === bulkCheckedIds.value.size
);
const noneCheckedForBulkUpdate = computed(
  () => bulkCheckedIds.value.size === 0
);
function toggleAllChecked() {
  if (allCheckedForBulkUpdate.value) {
    bulkCheckedIds.value.clear();
  } else {
    bulkCheckedIds.value = new Set(Object.keys(updates.value));
  }
}

function openBulkEditDialog() {
  isBulkDialogOpen.value = true;

  if (bulkLocation.value == undefined) {
    const selectedItems = Object.values(updates.value).filter(
      (item) =>
        bulkCheckedIds.value.has(item.asset.id) && item.estimatedLocation
    );
    const averageLocation = selectedItems.reduce<LatLng>(
      (acc, item) => {
        if (item.estimatedLocation) {
          acc.lat += item.estimatedLocation.lat;
          acc.lng += item.estimatedLocation.lng;
        }
        return acc;
      },
      { lat: 0, lng: 0 }
    );
    if (averageLocation.lng !== 0 && averageLocation.lat !== 0) {
      bulkLocation.value = {
        lat: averageLocation.lat / selectedItems.length,
        lng: averageLocation.lng / selectedItems.length,
      };
    }
  }
}

function commitBulkLocationEdits() {
  if (!bulkLocation.value) return;

  for (const item of Object.values(updates.value)) {
    if (bulkCheckedIds.value.has(item.asset.id)) {
      item.estimatedLocation = { ...bulkLocation.value };
      item.confirmEdit = true;
    }
  }

  bulkCheckedIds.value.clear(); // Reset selection for the next bulk edit
  isBulkDialogOpen.value = false;
  bulkLocation.value = undefined;
  bulkEditMode.value = false;
}

const loading = ref(false);
async function confirm({ hideRest = false } = {}) {
  loading.value = true;
  try {
    await Promise.all(
      confirmedUpdates.value.map((item) =>
        updateAsset({
          id: item.asset.id,
          updateAssetDto: {
            latitude: item.estimatedLocation!.lat,
            longitude: item.estimatedLocation!.lng,
          },
        })
      )
    );
  } catch (error) {
    alert("Error saving changes, check the console");
    console.error(error);
    return;
  } finally {
    loading.value = false;
  }

  const totalItemsCount = items.value.length;
  const confirmedUpdatesCount = confirmedUpdates.value.length;

  if (hideRest) {
    const unconfirmedUpdates = Object.values(updates.value).filter(
      (item) => !item.confirmEdit
    );
    for (const { asset } of unconfirmedUpdates) {
      alreadySeen.value.add(asset.id);
    }
  }

  clear();
  updates.value = {};
  // Reset back to the first page that would now contain unseen assets.
  page.value =
    Math.ceil((totalItemsCount - confirmedUpdatesCount) / pageSize) || 1;
  bulkCheckedIds.value.clear();

  alert(
    "Locations saved, but it may take some time for Immich to finish reverse-geocoding. Please reload the page if you see images you already confirmed"
  );
  setTimeout(() => {
    // Give some time for the reverse geo-coding to finish
    execute();
  }, 500);
}

function clearHidden() {
  const seenItems = allAssets.value.filter(({ asset }) =>
    alreadySeen.value.has(asset.id)
  );
  for (const { asset } of seenItems) {
    alreadySeen.value.delete(asset.id);
  }
}
</script>

<template>
  <Message v-if="error" severity="error">
    {{ error }}
  </Message>
  <section v-else>
    <div class="mb-2 flex gap-2 items-center sticky top-0 z-[10000] p-2 bg-primary-contrast">
      <ToggleButton
        v-model="bulkEditMode"
        on-label="Exit bulk edit mode"
        off-label="Enable bulk edit mode"
        :class="bulkEditMode ? 'p-button-secondary' : 'p-button-info'"
        :disabled="loading"
      />
      <template v-if="bulkEditMode">
        <Checkbox
          binary
          :disabled="loading"
          :indeterminate="!allCheckedForBulkUpdate && !noneCheckedForBulkUpdate"
          :model-value="allCheckedForBulkUpdate"
          @value-change="toggleAllChecked"
        />
        <span>{{ allCheckedForBulkUpdate ? "Uncheck" : "Check" }} all</span>
        <Button
          @click="openBulkEditDialog"
          :disabled="bulkCheckedIds.size === 0 || loading"
          severity="success"
        >
          Update selected assets' locations
        </Button>
      </template>
    </div>
    <div class="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-4">
      <article v-for="{ asset } in updates" :key="asset.id">
        <ImmichAsset
          :asset
          :disable-confirm="
            (!bulkEditMode && updates[asset.id].estimatedLocation == null) ||
            loading
          "
          :confirm-label="bulkEditMode ? 'Select for edit' : 'Confirm edit'"
          :confirm-edit="
            bulkEditMode
              ? bulkCheckedIds.has(asset.id)
              : updates[asset.id].confirmEdit
          "
          @update:confirm-edit="
            (val) => {
              if (bulkEditMode) {
                if (val) bulkCheckedIds.add(asset.id);
                else bulkCheckedIds.delete(asset.id);
              } else {
                updates[asset.id].confirmEdit = val;
              }
            }
          "
        >
          <Message
            v-if="timeline == null && !updates[asset.id].estimatedLocation"
            severity="info"
          >
            Timeline not uploaded. Cannot estimate location.
          </Message>
          <LeafletMap
            v-model="updates[asset.id].estimatedLocation"
            @update:model-value="updates[asset.id].confirmEdit = true"
            :segments="updates[asset.id].geometry"
          />
        </ImmichAsset>
      </article>
      <article v-if="items.length === 0" class="col-span-full">
        <Message>No images found</Message>
      </article>
    </div>
    <!-- Bulk location update dialog -->
    <Dialog
      v-model:visible="isBulkDialogOpen"
      header="Update location for selected images"
      modal
    >
      <div class="mb-2">
        <p>
          Pick a new location to apply to
          <b>{{ bulkCheckedIds.size }}</b> images:
        </p>
        <!-- TODO: pass segments of all selected images? -->
        <LeafletMap v-model="bulkLocation" style="height: 300px" />
      </div>
      <template #footer>
        <Button @click="isBulkDialogOpen = false">Cancel</Button>
        <Button
          @click="commitBulkLocationEdits"
          :disabled="bulkLocation == undefined"
          severity="success"
        >
          Update
        </Button>
      </template>
    </Dialog>
    <footer class="flex justify-end items-center gap-3 mt-3">
      <Message v-if="status !== 'success'">
        <p>{{ status }}...</p>
      </Message>
      <Message v-if="alreadySeenCount > 0" severity="secondary">
        <p>{{ alreadySeenCount }} assets hidden</p>
        <Button @click="clearHidden" :disabled="loading">
          Show hidden items
        </Button>
      </Message>
      <Button
        @click="page++"
        :disabled="!result?.hasNextPage || status === 'pending' || loading"
      >
        Load more items (showing {{ items.length }} / {{ (result?.total ?? 0) - alreadySeenCount }})
      </Button>
      <Button
        :disabled="confirmedUpdates.length === 0 || loading || bulkEditMode"
        @click="confirm()"
      >
        Save {{ confirmedUpdates.length }} change{{
          confirmedUpdates.length === 1 ? "" : "s"
        }}
      </Button>
      <Button
        :disabled="confirmedUpdates.length === 0 || loading || bulkEditMode"
        @click="confirm({ hideRest: true })"
      >
        Save changes and hide the rest
      </Button>
    </footer>
  </section>
</template>
