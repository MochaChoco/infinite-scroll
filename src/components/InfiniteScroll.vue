<template>
  <div class="root" ref="rootRef">
    <div class="container" ref="scrollContainerRef">
      <div ref="loadMoreTriggerTopRef" class="load-more-trigger-top"></div>
      <div v-for="(item, index) in visibleItems" :key="index" class="item">
        {{
          item.id +
          ". " +
          item.first_name +
          " " +
          item.last_name +
          " - " +
          item.email
        }}
      </div>
      <div ref="loadMoreTriggerBottomRef" class="load-more-trigger"></div>
      <div v-if="loading" class="loading">Loading...</div>
    </div>
  </div>
</template>

<script lang="ts" setup>
import { ref, onMounted, onUnmounted } from "vue";
import { DataType } from "@/types";

const totalItems = ref<DataType>({
  first: 1,
  prev: null,
  next: null,
  last: 0,
  pages: 0,
  items: 0,
  data: [],
});
const visibleItems = ref<DataType["data"]>([]);

const loadMoreTriggerTopRef = ref<HTMLElement | null>(null);
const loadMoreTriggerBottomRef = ref<HTMLElement | null>(null);
const rootRef = ref<HTMLElement | null>(null);
const scrollContainerRef = ref<HTMLElement | null>(null);

const loading = ref(false);
const page = ref(0);
const pageSize = 20;
const prevDireciton = ref<"up" | "down">("down");
const bufferSize = pageSize * 3; // 버퍼 크기를 3페이지 크기로 설정

const loadItems = async (direction: "up" | "down") => {
  if (loading.value) return;
  loading.value = true;

  try {
    const newPage =
      direction === "down"
        ? prevDireciton.value === "down"
          ? page.value + 1
          : page.value + 3
        : prevDireciton.value === "down"
        ? page.value - 3
        : page.value - 1;

    const response = await fetch(
      `http://localhost:3001/items?_page=${newPage}&_per_page=${pageSize}`
    );
    const data: DataType = await response.json();

    switch (direction) {
      case "down":
        // 스크롤 높이 추가
        if (page.value >= 1) {
          const item = document.getElementsByClassName("item")[0];
          const rootHeight =
            (page.value + 1) * pageSize * item.getClientRects()[0].height;
          rootRef.value!.style.height = rootHeight + "px";
        }
        if (visibleItems.value.length < bufferSize) {
          visibleItems.value.push(...data.data);
        } else {
          // 스크롤 위치 조정
          const item = document.getElementsByClassName("item")[0];
          scrollContainerRef.value!.style.top = "";
          scrollContainerRef.value!.style.bottom =
            (prevDireciton.value === "down" ? page.value - 2 : page.value) *
              -pageSize *
              item.getClientRects()[0].height +
            "px";

          // 새 데이터를 추가하고 가장 오래된 데이터를 제거
          visibleItems.value = [
            ...visibleItems.value.slice(pageSize),
            ...data.data,
          ];
        }
        break;
      case "up":
        if (visibleItems.value.length < bufferSize) {
          visibleItems.value.unshift(...data.data);
        } else {
          const item = document.getElementsByClassName("item")[0];

          // 스크롤 위치 조정
          scrollContainerRef.value!.style.bottom =
            Number(scrollContainerRef.value!.style.bottom.replace("px", "")) +
            pageSize * item.getClientRects()[0].height +
            "px";

          // 새 데이터를 추가하고 가장 최근의 데이터를 제거
          visibleItems.value = [
            ...data.data,
            ...visibleItems.value.slice(0, bufferSize - pageSize),
          ];
        }
        break;
    }

    page.value = newPage;
    totalItems.value = data;
    prevDireciton.value = direction;
  } catch (error) {
    console.error("Failed to load items:", error);
  } finally {
    loading.value = false;
  }
};

const onIntersect = (
  entries: IntersectionObserverEntry[],
  observer: IntersectionObserver
) => {
  if (entries[0].isIntersecting) {
    if (observer === observerBottom) {
      loadItems("down");
    } else {
      loadItems("up");
    }
  }
};

const observerTop = new IntersectionObserver((entries) => {
  if (page.value > 1) onIntersect(entries, observerTop);
});
const observerBottom = new IntersectionObserver((entries) => {
  if (page.value < totalItems.value.last) onIntersect(entries, observerBottom);
});

onMounted(async () => {
  if (loadMoreTriggerTopRef.value) {
    observerTop.observe(loadMoreTriggerTopRef.value);
  }
  if (loadMoreTriggerBottomRef.value) {
    observerBottom.observe(loadMoreTriggerBottomRef.value);
  }
  loadItems("down");
});

onUnmounted(() => {
  if (loadMoreTriggerTopRef.value) {
    observerTop.unobserve(loadMoreTriggerTopRef.value);
  }
  if (loadMoreTriggerBottomRef.value) {
    observerBottom.unobserve(loadMoreTriggerBottomRef.value);
  }
});
</script>

<style>
.container {
  position: relative;
  max-width: 600px;
  margin: 0 auto;
  padding: 20px;
}

.item {
  padding: 10px;
  margin: 0px 0;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.load-more-trigger-top,
.load-more-trigger-bottom {
  height: 20px;
}

.loading {
  text-align: center;
  margin: 20px 0;
}
</style>
