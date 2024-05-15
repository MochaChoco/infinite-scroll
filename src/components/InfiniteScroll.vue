<template>
  <div class="root" ref="rootRef">
    <div class="scrollContainer" ref="scrollContainerRef">
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

const multiples = 3;
const loading = ref(false);
const page = ref(0); // mount 되면서 1로 변경됨.
const pageSize = 20;
const prevDireciton = ref<"up" | "down">("down"); // 이전에 어느 방향에서 데이터를 불러왔는지 저장한다.
const bufferSize = pageSize * multiples; // 화면에 표시할 총 아이템의 갯수. 화면에 보이는 것과 화면 위와 아래에 표시할 것을 고려하여 버퍼 크기를 크기의 3배로 설정한다.

// 스크롤 처리 함수
const loadItems = async (direction: "up" | "down") => {
  if (loading.value) return;
  loading.value = true;

  try {
    // 이전의 스크롤 방향과 현재 방향을 비교하여 올바른 page 번호를 계산함
    const newPage =
      direction === "down"
        ? prevDireciton.value === "down"
          ? page.value + 1
          : page.value + multiples
        : prevDireciton.value === "down"
        ? page.value - multiples
        : page.value - 1;

    const response = await fetch(
      `http://localhost:3001/items?_page=${newPage}&_per_page=${pageSize}`
    );
    const data: DataType = await response.json();

    switch (direction) {
      case "down":
        // 화면을 내리면 스크롤 높이를 추가하는 로직
        if (page.value >= 1) {
          const item = document.getElementsByClassName("item")[0];
          const rootHeight =
            (page.value + 1) * pageSize * item.getClientRects()[0].height;
          rootRef.value!.style.height = rootHeight + "px";
        }

        // 현재 아이템들과 bufferSize가 일치하는지 체크하여 데이터를 삽입할지 대체할지 분기 처리함.
        if (visibleItems.value.length < bufferSize) {
          visibleItems.value.push(...data.data);
        } else {
          // 리스트 위치 조정
          const item = document.getElementsByClassName("item")[0];
          scrollContainerRef.value!.style.top = "";
          scrollContainerRef.value!.style.bottom =
            (prevDireciton.value === "down"
              ? page.value - (multiples - 1)
              : page.value) *
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

          // 리스트 위치 조정
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
    prevDireciton.value = direction; // 현재 스크롤 위치를 저장하여 다음 로드때 사용함
  } catch (error) {
    console.error("Failed to load items:", error);
  } finally {
    loading.value = false;
  }
};

// 호출된 observer에 따른 분기 처리
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

// .loadMoreTriggerTop이 화면에 들어왔을 때 (스크롤을 위로 올렸을 때)
const observerTop = new IntersectionObserver((entries) => {
  if (page.value > 1) onIntersect(entries, observerTop);
});

// .loadMoreTriggerBottom이 화면에 들어왔을 때 (스크롤을 아래로 내렸을 때)
const observerBottom = new IntersectionObserver((entries) => {
  if (page.value < totalItems.value.last) onIntersect(entries, observerBottom);
});

// mount하면 observer를 등록하고, 데이터를 로드한다.
onMounted(async () => {
  if (loadMoreTriggerTopRef.value) {
    observerTop.observe(loadMoreTriggerTopRef.value);
  }
  if (loadMoreTriggerBottomRef.value) {
    observerBottom.observe(loadMoreTriggerBottomRef.value);
  }
  loadItems("down");
});

// unmount하면 지정된 observer를 해제하여 불필요한 메모리 낭비를 막는다.
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
.scrollContainer {
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
