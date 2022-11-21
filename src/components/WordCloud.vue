<script setup>
import * as d3 from 'd3'
import * as cloud from 'd3-cloud'
import { ref, computed, onMounted, onUnmounted, watch } from 'vue'
import { storeToRefs } from 'pinia'

import { useDatasetsStore } from '@/stores/datasets.js';
import { toFloat } from '@/helpers/mathHelper.js';
import { throttle } from '@/helpers/logicHelper.js';

const store = useDatasetsStore()
const { datasetSamplesPrepared } = storeToRefs(store)

const counter = ref(0);
const wordcloud = ref(null);

const datasetSamples = computed(() => {
   return counter.value + JSON.stringify(datasetSamplesPrepared.value)
})

function incCounter(){
   ++counter.value
}

const debounceIncCounter = throttle(incCounter, 150);

function drawCloud() {
   d3.select(wordcloud.value).selectAll('*').remove();

   if(!datasetSamplesPrepared.value){
      return
   }

   const samples = (datasetSamplesPrepared.value || []).sort((a, b) => a.weight - b.weight),
      words = (samples || []).map(function (d) { return { text: d.label, size: d.weight }; }),
      expected = samples.length,
      minWeight = samples.at(0).weight,
      maxWeight = samples.at(-1).weight;

   const minFont = 5, maxFont = 25;
   const rootDomRect = wordcloud.value.getBoundingClientRect()

   const fontSize = d3.scaleLinear().range([minFont, maxFont]).clamp(true)
   samples.length && fontSize.domain([+minWeight || 1, +maxWeight])


   let scale, wordScale = 1, tries = 0, arr = [], tryCount = 3

   const width = Math.floor(rootDomRect.width - 20),
      height = Math.floor(rootDomRect.height - 20)

   const svg = d3.select(wordcloud.value).append("svg")
      .attr("width", width)
      .attr("height", height)
      .append("g")
      .attr("transform", "scale(" + 1 + ")translate(" + [width >> 1, height >> 1] + ")")

   const layout = cloud()
      .spiral('rectangular')
      .size([width, height])
      .words(words)
      .random(function (d) { return 0.5; })
      .padding(0)
      .rotate(0)
      .fontSize(function (t) {
         return fontSize(+t.size) * wordScale
      })
      .on("end", draw);

   layout.start();

   function draw(words, e) {
      if (words.length < expected) {
         arr.push(words.length)
         tries++;
         layout.stop();
         if (tries === tryCount) {
            const avg = arr.reduce((prev, next) => { return prev + next }, 0) / tryCount
            wordScale *= avg / expected
            wordScale *= 0.8
            tries = 0
            arr = []
         }
         layout.start();
         return;
      }

      scale = e
         ? Math.min(width / Math.abs(e[1].x - width / 2), width / Math.abs(e[0].x - width / 2), height / Math.abs(e[1].y - height / 2), height / Math.abs(e[0].y - height / 2)) / 2
         : 1

      svg
         .selectAll("text")
         .data(words)
         .enter()
         .append("text")
         .style("font-size", function (d) { return d.size; })
         .style("fill", function (d, i) { return d3.schemeCategory10[i % 10]; })
         .attr("text-anchor", "middle")
         .style("font-family", "Impact")
         .attr("transform", function (d) {
            return "translate(" + [d.x, d.y] + ")rotate(" + d.rotate + ")";
         })
         .text(function (d) { return d.text; });

      const svgGDom = wordcloud.value.querySelector('svg > g'), svgGDomRect = svgGDom.getBoundingClientRect()
      const Xscale = width / svgGDomRect.width, Yscale = height / svgGDomRect.height;
      scale = toFloat(Math.min(Xscale, Yscale), 3)

      const partOfScale = scale / 1000;
      if (scale === toFloat(Xscale, 3)) {
         while (svgGDomRect.width * scale > width) {
            scale -= partOfScale;
         }
      } else {
         while (svgGDomRect.height * scale > height) {
            scale -= partOfScale;
         }
      }
      svg
      // .transition().delay(100).duration(1e3)
      .attr("transform", "translate(" + [width >> 1, height >> 1] + ")scale(" + scale * 0.9 + ")")
   }
}

onMounted(() => {
   window.addEventListener('resize', debounceIncCounter);
});

onUnmounted(() => {
   window.removeEventListener('resize', debounceIncCounter);
});


watch(datasetSamples, () => {
   drawCloud();
})
</script>


<template>
   <div class="wordcloud" ref="wordcloud"></div>
</template>

<style lang="scss" scoped>
.wordcloud {
   width: 100%;
   height: 100%;

   border: 1px solid black;
   box-sizing: border-box;

   display: flex;
   flex-direction: column;
   justify-content: center;
   align-items: center;
}
</style>