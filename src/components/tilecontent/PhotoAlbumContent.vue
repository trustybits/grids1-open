<template>
  <!-- Tile card-stack view -->
  <div
    class="album-tile"
    ref="tileRef"
    @mousemove="onTileMouseMove"
    @mouseleave="onTileMouseLeave"
  >
    <!-- Empty state -->
    <div v-if="isEmpty" class="album-empty">
      <svg class="album-empty-icon" width="40" height="40" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round">
        <rect x="2" y="6" width="20" height="16" rx="2"/>
        <rect x="6" y="2" width="14" height="16" rx="2" opacity="0.5"/>
        <rect x="10" y="0" width="10" height="12" rx="2" opacity="0.25"/>
        <circle cx="11" cy="14" r="2"/>
        <polyline points="19 22 15 17 9 22"/>
      </svg>
      <span class="album-empty-label">Photo Album</span>
    </div>

    <!-- Card stack (non-empty) -->
    <template v-else>
      <!-- Back cards — visible as slivers beneath the top card -->
      <div
        v-for="n in backCardCount"
        :key="`back-${n}`"
        class="album-card album-card--back"
        :style="backCardStyle(n)"
        aria-hidden="true"
      >
        <img
          v-if="localItems[n]?.type === 'image'"
          :src="localItems[n].src"
          class="album-card-media"
          draggable="false"
        />
        <video
          v-else-if="localItems[n]?.src"
          :src="localItems[n].src"
          class="album-card-media"
          muted
          preload="none"
        />
      </div>

      <!-- Top card -->
      <div class="album-card album-card--top" :style="topCardStyle">
        <img
          v-if="localItems[0]?.type === 'image'"
          :src="localItems[0].src"
          class="album-card-media"
          draggable="false"
        />
        <video
          v-else-if="localItems[0]?.src"
          :src="localItems[0].src"
          class="album-card-media"
          muted
          preload="metadata"
        />
        <!-- Count badge -->
        <div v-if="localItems.length > 1" class="album-count-badge">
          {{ localItems.length }}
        </div>
      </div>
    </template>
  </div>

  <!-- Lightbox — teleported to document body -->
  <Teleport to="body">
    <div
      v-if="isOpen"
      class="album-lightbox"
      :class="{ 'is-closing': isClosing }"
      ref="lightboxRootRef"
      tabindex="-1"
      @keydown="onKeyDown"
    >
      <!-- Scrim -->
      <div class="album-scrim" @click="closeLightbox" />

      <!-- Slinky item stack -->
      <div
        class="album-stage"
        :style="stageStyle"
        @wheel.prevent="onWheel"
        @click.self="closeLightbox"
      >
        <div
          v-for="(item, idx) in localItems"
          :key="item.id"
          class="album-stage-item"
          :class="stageItemClass(idx)"
          :style="stageItemStyle(idx)"
        >
          <img
            v-if="item.type === 'image'"
            :src="item.src"
            class="album-stage-media"
            draggable="false"
            @click.stop
          />
          <video
            v-else
            :src="item.src"
            class="album-stage-media"
            :controls="idx === currentIndex"
            :autoplay="idx === currentIndex"
            playsinline
            @click.stop
          />
        </div>
      </div>

      <!-- Navigation arrows -->
      <button
        v-if="currentIndex > 0"
        class="album-nav album-nav--prev"
        aria-label="Previous"
        @click.stop="navigate(-1)"
      >
        <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
          <polyline points="15 18 9 12 15 6"/>
        </svg>
      </button>
      <button
        v-if="currentIndex < localItems.length - 1"
        class="album-nav album-nav--next"
        aria-label="Next"
        @click.stop="navigate(1)"
      >
        <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
          <polyline points="9 18 15 12 9 6"/>
        </svg>
      </button>

      <!-- Counter pill -->
      <div v-if="localItems.length > 1" class="album-counter">
        {{ currentIndex + 1 }} / {{ localItems.length }}
      </div>

      <!-- Add photos button (owner only) -->
      <button
        v-if="layoutStore.isOwner && layoutStore.canEdit"
        class="album-add-btn"
        :class="{ 'is-uploading': isUploading }"
        :disabled="isUploading"
        @click.stop="triggerAddMedia"
        aria-label="Add photos or videos"
      >
        <svg v-if="!isUploading" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round">
          <line x1="12" y1="5" x2="12" y2="19"/>
          <line x1="5" y1="12" x2="19" y2="12"/>
        </svg>
        <span>{{ isUploading ? `${uploadProgress}%` : 'Add photos' }}</span>
      </button>

      <!-- Hidden file input -->
      <input
        ref="fileInputRef"
        type="file"
        accept="image/*,video/*"
        multiple
        style="display: none"
        @change="onFilesSelected"
      />
    </div>
  </Teleport>
</template>

<script lang="ts">
import {
  defineComponent,
  ref,
  computed,
  watch,
  onMounted,
  onUnmounted,
  nextTick,
  inject,
} from "vue";
import { type PhotoAlbumContent, type PhotoAlbumItem } from "@/types/TileContent";
import { useLayoutStore } from "@/stores/layout";
import { useFileUpload } from "@/composables/useFileUpload";
import { useColorPicker } from "@/composables/useColorPicker";

function makeId(): string {
  return `${Date.now()}-${Math.random().toString(36).slice(2, 9)}`;
}

export default defineComponent({
  name: "PhotoAlbumContent",
  emits: ["background-color-change", "text-color-change"],
  props: {
    content: {
      type: Object as () => PhotoAlbumContent,
      required: true,
    },
  },
  setup(props, { emit }) {
    const layoutStore = useLayoutStore();
    const { uploadFileToUrl } = useFileUpload();
    const tileId = inject<string>("tileId", "");

    useColorPicker(tileId, props.content, emit, "background");

    // ── Local item state ──────────────────────────────────────────────
    // We keep a local copy so we can show blob-URL previews during upload
    // without waiting for Firestore round-trips.
    const localItems = ref<PhotoAlbumItem[]>([...props.content.items]);

    watch(
      () => props.content.items,
      (incoming) => {
        // Merge permanent URLs back in for any item that's still a blob
        const merged = incoming.map((serverItem) => {
          const local = localItems.value.find((l) => l.id === serverItem.id);
          // Keep the local entry if it has a blob URL that hasn't been replaced yet
          if (local && local.src.startsWith("blob:")) return local;
          return serverItem;
        });
        // Append local-only items (pending uploads not yet in content)
        localItems.value.forEach((local) => {
          if (!merged.find((m) => m.id === local.id)) merged.push(local);
        });
        localItems.value = merged;
      },
      { deep: true },
    );

    const isEmpty = computed(() => localItems.value.length === 0);
    const backCardCount = computed(() => Math.min(localItems.value.length - 1, 2));

    // ── Hover mouse tracking ──────────────────────────────────────────
    const tileRef = ref<HTMLElement | null>(null);
    const mouseNorm = ref({ x: 0, y: 0 }); // –1..1 relative to tile center
    const isHovering = ref(false);

    const onTileMouseMove = (e: MouseEvent) => {
      if (!tileRef.value || isEmpty.value) return;
      const rect = tileRef.value.getBoundingClientRect();
      mouseNorm.value = {
        x: (e.clientX - rect.left - rect.width / 2) / (rect.width / 2),
        y: (e.clientY - rect.top - rect.height / 2) / (rect.height / 2),
      };
      isHovering.value = true;
    };

    const onTileMouseLeave = () => {
      isHovering.value = false;
      mouseNorm.value = { x: 0, y: 0 };
    };

    // Top card: tilts toward mouse, lifts slightly
    const topCardStyle = computed(() => {
      if (!isHovering.value) {
        return { transform: "perspective(600px) rotateX(0deg) rotateY(0deg) translateZ(0px)" };
      }
      const rx = -mouseNorm.value.y * 6;
      const ry = mouseNorm.value.x * 8;
      return {
        transform: `perspective(600px) rotateX(${rx}deg) rotateY(${ry}deg) translateZ(10px)`,
        transition: "transform 0.15s ease-out",
      };
    });

    // Back cards: fan out toward the opposite side of the mouse
    const backCardStyle = (n: number) => {
      const depth = n; // 1 or 2
      if (!isHovering.value) {
        // Resting: slight cascade offset
        return {
          transform: `translate(${depth * 5}px, ${depth * -5}px) scale(${1 - depth * 0.04})`,
          opacity: 1 - depth * 0.15,
          zIndex: 2 - depth,
        };
      }
      // Hover: fan out away from mouse, like a deck spring
      const mx = mouseNorm.value.x;
      const my = mouseNorm.value.y;
      const fanX = -mx * depth * 22;
      const fanY = -my * depth * 14;
      const fanRot = -mx * depth * 10;
      return {
        transform: `translate(${fanX}px, ${fanY}px) rotate(${fanRot}deg) scale(${1 - depth * 0.06})`,
        opacity: 1 - depth * 0.2,
        zIndex: 2 - depth,
        transition: "transform 0.2s cubic-bezier(0.34,1.56,0.64,1), opacity 0.2s ease",
      };
    };

    // ── Lightbox state ────────────────────────────────────────────────
    const isOpen = ref(false);
    const isClosing = ref(false);
    const currentIndex = ref(0);
    const lightboxRootRef = ref<HTMLElement | null>(null);
    const stageFromX = ref(0);
    const stageFromY = ref(0);
    const stageFromScale = ref(0.15);

    const openLightbox = () => {
      if (isEmpty.value) {
        triggerAddMedia();
        return;
      }

      // Capture tile rect so we can animate from that position
      if (tileRef.value) {
        const rect = tileRef.value.getBoundingClientRect();
        const vw = window.innerWidth;
        const vh = window.innerHeight;
        stageFromX.value = rect.left + rect.width / 2 - vw / 2;
        stageFromY.value = rect.top + rect.height / 2 - vh / 2;
        const maxW = Math.min(vw * 0.88, 900);
        const maxH = Math.min(vh * 0.88, 700);
        stageFromScale.value = Math.min(rect.width / maxW, rect.height / maxH, 0.25);
      }

      currentIndex.value = 0;
      isOpen.value = true;
      nextTick(() => lightboxRootRef.value?.focus());
    };

    const closeLightbox = () => {
      if (isClosing.value) return;
      isClosing.value = true;
      setTimeout(() => {
        isOpen.value = false;
        isClosing.value = false;
      }, 380);
    };

    const stageStyle = computed(() => ({
      "--from-x": `${stageFromX.value}px`,
      "--from-y": `${stageFromY.value}px`,
      "--from-scale": stageFromScale.value,
    }));

    // ── Item classes & styles ─────────────────────────────────────────
    const stageItemClass = (idx: number) => {
      const delta = idx - currentIndex.value;
      if (delta === 0) return "is-active";
      if (delta < 0) return "is-past";
      if (delta === 1) return "is-next-1";
      if (delta === 2) return "is-next-2";
      return "is-hidden";
    };

    const stageItemStyle = (idx: number) => ({
      "--item-delay": `${Math.abs(idx - currentIndex.value) * 55}ms`,
    });

    // ── Navigation ────────────────────────────────────────────────────
    const navigate = (dir: -1 | 1) => {
      const next = currentIndex.value + dir;
      if (next >= 0 && next < localItems.value.length) {
        currentIndex.value = next;
      }
    };

    const onWheel = (e: WheelEvent) => {
      if (e.deltaY > 0) navigate(1);
      else navigate(-1);
    };

    const onKeyDown = (e: KeyboardEvent) => {
      if (!isOpen.value) return;
      if (e.key === "ArrowLeft" || e.key === "ArrowUp") { e.preventDefault(); navigate(-1); }
      else if (e.key === "ArrowRight" || e.key === "ArrowDown") { e.preventDefault(); navigate(1); }
      else if (e.key === "Escape") closeLightbox();
    };

    // ── Upload ────────────────────────────────────────────────────────
    const isUploading = ref(false);
    const uploadProgress = ref(0);
    const fileInputRef = ref<HTMLInputElement | null>(null);

    const triggerAddMedia = () => {
      fileInputRef.value?.click();
    };

    const onFilesSelected = async (event: Event) => {
      const input = event.target as HTMLInputElement;
      const files = Array.from(input.files || []);
      input.value = "";
      if (!files.length) return;

      isUploading.value = true;
      uploadProgress.value = 0;

      const blobUrls: string[] = [];

      for (let i = 0; i < files.length; i++) {
        const file = files[i];
        const isImage = file.type.startsWith("image/");
        const blobUrl = URL.createObjectURL(file);
        blobUrls.push(blobUrl);

        const id = makeId();
        const preview: PhotoAlbumItem = {
          id,
          src: blobUrl,
          type: isImage ? "image" : "video",
        };

        // Add preview immediately for instant feedback
        localItems.value = [...localItems.value, preview];

        try {
          const url = await uploadFileToUrl(file);
          // Swap blob URL with permanent Firebase URL
          localItems.value = localItems.value.map((item) =>
            item.id === id ? { ...item, src: url } : item,
          );
        } catch (err) {
          console.error("Photo album upload failed:", err);
          // Remove failed preview
          localItems.value = localItems.value.filter((item) => item.id !== id);
          URL.revokeObjectURL(blobUrl);
        }

        uploadProgress.value = Math.round(((i + 1) / files.length) * 100);
      }

      // Persist final list to Firestore
      layoutStore.patchTileContent(tileId, { items: localItems.value });
      layoutStore.updateLayout();

      // Clean up blob URLs (browser holds refs until revoked)
      blobUrls.forEach((u) => URL.revokeObjectURL(u));

      isUploading.value = false;
      uploadProgress.value = 0;
    };

    // ── Exposed for GridTile ──────────────────────────────────────────
    const onShortClick = () => {
      openLightbox();
    };

    // Cleanup on unmount
    onUnmounted(() => {
      // If lightbox is open and component unmounts, revoke any blob URLs
      localItems.value.forEach((item) => {
        if (item.src.startsWith("blob:")) URL.revokeObjectURL(item.src);
      });
    });

    return {
      layoutStore,
      tileRef,
      localItems,
      isEmpty,
      backCardCount,
      topCardStyle,
      backCardStyle,
      onTileMouseMove,
      onTileMouseLeave,

      isOpen,
      isClosing,
      currentIndex,
      lightboxRootRef,
      stageStyle,
      stageItemClass,
      stageItemStyle,

      navigate,
      onWheel,
      onKeyDown,
      closeLightbox,

      isUploading,
      uploadProgress,
      fileInputRef,
      triggerAddMedia,
      onFilesSelected,

      onShortClick,
    };
  },
});
</script>

<style scoped lang="scss">
/* ── Tile ─────────────────────────────────────────────────────────── */
.album-tile {
  width: 100%;
  height: 100%;
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: visible;
  cursor: pointer;
}

/* ── Empty state ──────────────────────────────────────────────────── */
.album-empty {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 10px;
  height: 100%;
  color: var(--color-content-low);
  user-select: none;
  pointer-events: none;
}

.album-empty-icon {
  opacity: 0.5;
}

.album-empty-label {
  font-size: 0.8rem;
  font-weight: 500;
  opacity: 0.6;
}

/* ── Card stack ───────────────────────────────────────────────────── */
.album-card {
  position: absolute;
  inset: 0;
  border-radius: calc(var(--tile-border-radius) - 2px);
  overflow: hidden;
  transition: transform 0.25s cubic-bezier(0.34, 1.56, 0.64, 1), opacity 0.2s ease;
  transform-origin: center center;
  will-change: transform;
}

.album-card--top {
  z-index: 3;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.2);
}

.album-card--back {
  pointer-events: none;
}

.album-card-media {
  width: 100%;
  height: 100%;
  object-fit: cover;
  display: block;
  pointer-events: none;
  user-select: none;
}

/* Count badge */
.album-count-badge {
  position: absolute;
  bottom: 10px;
  right: 10px;
  background: rgba(0, 0, 0, 0.55);
  backdrop-filter: blur(6px);
  color: #fff;
  font-size: 11px;
  font-weight: 600;
  letter-spacing: 0.02em;
  padding: 3px 8px;
  border-radius: 20px;
  pointer-events: none;
  user-select: none;
  z-index: 4;
}

/* ── Lightbox root ────────────────────────────────────────────────── */
.album-lightbox {
  position: fixed;
  inset: 0;
  z-index: 9000;
  display: flex;
  align-items: center;
  justify-content: center;
  outline: none;
}

/* ── Scrim ────────────────────────────────────────────────────────── */
.album-scrim {
  position: absolute;
  inset: 0;
  background: rgba(0, 0, 0, 0.72);
  backdrop-filter: blur(10px) brightness(0.55);
  animation: scrimIn 0.35s ease-out forwards;

  .album-lightbox.is-closing & {
    animation: scrimOut 0.38s ease-in forwards;
  }
}

@keyframes scrimIn {
  from { opacity: 0; backdrop-filter: blur(0px) brightness(1); }
  to   { opacity: 1; backdrop-filter: blur(10px) brightness(0.55); }
}

@keyframes scrimOut {
  from { opacity: 1; backdrop-filter: blur(10px) brightness(0.55); }
  to   { opacity: 0; backdrop-filter: blur(0px) brightness(1); }
}

/* ── Stage (item container) ───────────────────────────────────────── */
.album-stage {
  position: relative;
  z-index: 1;
  width: min(88vw, 900px);
  height: min(88vh, 700px);
  display: flex;
  align-items: center;
  justify-content: center;

  /* Entry: slide in from tile position */
  animation: stageOpen 0.5s cubic-bezier(0.34, 1.56, 0.64, 1) forwards;

  .album-lightbox.is-closing & {
    animation: stagePop 0.38s cubic-bezier(0.55, 0, 1, 0.45) forwards;
  }
}

@keyframes stageOpen {
  from {
    opacity: 0;
    transform: translate(var(--from-x), var(--from-y)) scale(var(--from-scale));
  }
  to {
    opacity: 1;
    transform: translate(0, 0) scale(1);
  }
}

@keyframes stagePop {
  0%   { transform: scale(1);    opacity: 1; }
  28%  { transform: scale(1.07); opacity: 1; }
  100% { transform: scale(0);    opacity: 0; }
}

/* ── Stage items ──────────────────────────────────────────────────── */
.album-stage-item {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  transition:
    transform 0.45s cubic-bezier(0.34, 1.56, 0.64, 1),
    opacity 0.3s ease;
  /* slinky: each item animates in with a small delay based on its stack distance */
  transition-delay: var(--item-delay, 0ms);
  pointer-events: none;
  border-radius: 12px;
  overflow: hidden;
}

/* Active item: full size, centered, fully visible */
.album-stage-item.is-active {
  opacity: 1;
  transform: scale(1) translateY(0);
  z-index: 10;
  pointer-events: auto;
  box-shadow: 0 16px 48px rgba(0, 0, 0, 0.45);
}

/* Next item in queue: peeking behind active, slinky trailing */
.album-stage-item.is-next-1 {
  opacity: 0.65;
  transform: scale(0.92) translateY(22px);
  z-index: 9;
}

.album-stage-item.is-next-2 {
  opacity: 0.35;
  transform: scale(0.84) translateY(42px);
  z-index: 8;
}

/* Items already navigated past or too far away */
.album-stage-item.is-past,
.album-stage-item.is-hidden {
  opacity: 0;
  transform: scale(0.8) translateY(-30px);
  z-index: 7;
  pointer-events: none;
}

/* ── Stage media ──────────────────────────────────────────────────── */
.album-stage-media {
  max-width: 100%;
  max-height: 100%;
  width: auto;
  height: auto;
  object-fit: contain;
  border-radius: 10px;
  display: block;
  user-select: none;
  pointer-events: none;
}

.album-stage-item.is-active .album-stage-media {
  pointer-events: auto;
}

/* ── Navigation arrows ────────────────────────────────────────────── */
.album-nav {
  position: absolute;
  top: 50%;
  transform: translateY(-50%);
  z-index: 20;
  width: 44px;
  height: 44px;
  border-radius: 50%;
  border: none;
  background: rgba(255, 255, 255, 0.15);
  backdrop-filter: blur(8px);
  color: #fff;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: background 0.15s ease, transform 0.15s ease;

  &:hover {
    background: rgba(255, 255, 255, 0.28);
    transform: translateY(-50%) scale(1.08);
  }

  &:active {
    transform: translateY(-50%) scale(0.95);
  }
}

.album-nav--prev { left: clamp(16px, 3vw, 48px); }
.album-nav--next { right: clamp(16px, 3vw, 48px); }

/* ── Counter ──────────────────────────────────────────────────────── */
.album-counter {
  position: absolute;
  bottom: clamp(16px, 3vh, 32px);
  left: 50%;
  transform: translateX(-50%);
  z-index: 20;
  background: rgba(0, 0, 0, 0.45);
  backdrop-filter: blur(6px);
  color: rgba(255, 255, 255, 0.85);
  font-size: 12px;
  font-weight: 500;
  padding: 5px 14px;
  border-radius: 20px;
  pointer-events: none;
  user-select: none;
  letter-spacing: 0.04em;
}

/* ── Add photos button ────────────────────────────────────────────── */
.album-add-btn {
  position: absolute;
  bottom: clamp(16px, 3vh, 32px);
  right: clamp(16px, 3vw, 48px);
  z-index: 20;
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 8px 14px;
  border-radius: 20px;
  border: 1px solid rgba(255, 255, 255, 0.25);
  background: rgba(255, 255, 255, 0.12);
  backdrop-filter: blur(8px);
  color: rgba(255, 255, 255, 0.9);
  font-size: 12px;
  font-weight: 500;
  cursor: pointer;
  transition: background 0.15s ease, border-color 0.15s ease;

  &:hover:not(:disabled) {
    background: rgba(255, 255, 255, 0.22);
    border-color: rgba(255, 255, 255, 0.4);
  }

  &:disabled,
  &.is-uploading {
    opacity: 0.65;
    cursor: default;
  }
}
</style>
