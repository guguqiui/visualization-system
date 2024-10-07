<template>
  <div class="container">
    <!-- 图表部分 -->
    <div id="chart" class="chart">
    </div>
    <div class="filter">
        <label for="typeSelect">Select Type:</label>
        <select id="typeSelect" v-model="selectedType" @change="filterByType">
          <option value="">All Types</option>
          <option v-for="type in availableTypes" :key="type" :value="type">{{ type }}</option>
        </select>
      </div>
    
    <!-- 展示点击的 instance 列表的区域 -->
    <div class="instance-content" v-if="selectedInstances.length > 0">
      <h3>Related Instance ({{ currentIndex + 1 }} / {{ selectedInstances.length }})</h3>
      
      <!-- 当前 instance 展示 -->
      <div class="instance-item">
        <p><strong>ID:</strong> {{ selectedInstances[currentIndex].instance_id }}</p>
        <p><strong>Question:</strong> {{ selectedInstances[currentIndex].question }}</p>
        <p><strong>Opinion:</strong> {{ selectedInstances[currentIndex].opinion }}</p>
        <p><strong>Category:</strong> {{ selectedInstances[currentIndex].category_ }}</p>
        <p><strong>Attribution:</strong> {{ selectedInstances[currentIndex].attribution_type_ }}</p>
        <p><strong>Type:</strong> {{ selectedInstances[currentIndex].type }}</p>
      </div>

      <!-- 翻页按钮 -->
      <div class="pagination">
        <button @click="prevInstance" :disabled="currentIndex === 0">Previous</button>
        <button @click="nextInstance" :disabled="currentIndex === selectedInstances.length - 1">Next</button>
      </div>
    </div>
  </div>
</template>

<script>
import * as echarts from 'echarts';

export default {
  name: 'App',
  data() {
    return {
      selectedInstances: [], // 存储当前选中的 instance 列表
      currentIndex: 0, // 当前展示的 instance 索引
      thirdElementToInstances: {} // 存储 thirdElement 到多个 instance 的映射
    };
  },
  mounted() {
    var chartDom = document.getElementById('chart');
    var myChart = echarts.init(chartDom);

    // type 颜色定义
    const typeToColorMap = {
      'cognitive judgment': '#FF6666',   // 红色
      'signaling event': '#66CC66',      // 浅绿色
      'nature': '#FFCC66',               // 浅橙色
      'suggestion': '#66CCCC',           // 浅青色
      'belief': '#9966CC',               // 紫色
      'feeling': '#FF99CC',              // 粉红色
      'situation': '#FF9966',            // 橙色
      'past experience': '#6699FF',      // 蓝色
      'potential outcome': '#FFCC99',    // 浅黄色
      'behavior': '#CC66FF',             // 浅紫色
      'intention': '#FF6699'             // 玫红色
    };

    // attribution 颜色定义
    const customColors = {
      'responsibility': '#F5B7B1',
      'fear and dangerousness': '#D7BDE2',
      'anger': '#D7BDE2',
      'no pity': '#D7BDE2',
      'social distance': '#ADD8E6',
      'no help': '#ADD8E6',
      'coercion segregation': '#ADD8E6',
      'no responsibility': '#D98880',
      'no fear and dangerousness': '#BB8FCE',
      'no anger': '#BB8FCE',
      'pity': '#BB8FCE',
      'help': '#87CEEB',
      'no social distance': '#87CEEB',
      'no coercion segregation': '#87CEEB'
    };

    const stigmaNodes = new Set([
      'responsibility', 'fear and dangerousness', 'anger', 
      'no pity', 'social distance', 
      'no help', 'coercion segregation'
    ]);

    const noStigmaNodes = new Set([
      'no responsibility', 'no fear and dangerousness', 
      'no anger', 'pity', 'help', 
      'no social distance', 'no coercion segregation'
    ]);

    const processedEdges = new Set();

    fetch('/data/test.json')
      .then(response => response.json())
      .then(data => {
        let nodes = [];
        let edges = [];

        const fixedNodePositions = {
          'responsibility': { x: 0, y: -1000 },
          'fear and dangerousness': { x: -500, y: -800 },
          'anger': { x: -700, y: -400 },
          'no pity': { x: -1000, y: 0 },
          'social distance': { x: -700, y: 400 },
          'no help': { x: -500, y: 800 },
          'coercion segregation': { x: 0, y: 1000 },
          'no responsibility': { x: 1000, y: -1000 },
          'no fear and dangerousness': { x: 1500, y: -800 },
          'no anger': { x: 1700, y: -400 },
          'pity': { x: 2000, y: 0 },
          'help': { x: 1700, y: 400 },
          'no social distance': { x: 1500, y: 800 },
          'no coercion segregation': { x: 1000, y: 1000 },
          'stigma': { x: 0, y: 0 }, // 固定 stigma 节点
          'no stigma': { x: 1000, y: 0 } // 固定 no stigma 节点
        };

        data.forEach(instance => {
          let source = instance.category_;
          let target = instance.attribution_type_;

          // 处理source节点
          // 为stigma或no stigma节点反转方向
          if (stigmaNodes.has(source) || noStigmaNodes.has(source)) {
            edges.push({
              source: target,
              target: source,
              label: 'related to',
              lineStyle: { color: customColors[source] || '#cccccc' }
            });
          } else {
            edges.push({
              source: target,
              target: source,
              label: 'related to',
              lineStyle: { color: customColors[target] || '#cccccc' }
            });
          }

          if (!nodes.some(node => node.name === source)) {
            let size = 100;
            let color = '#ff6666';
            let category = 'stigma';

            if (source === 'no stigma') {
              size = 100;
              color = '#66cc66';
              category = 'no stigma';
            }
            // 检查是否需要固定位置
            let fixed = false;
            let x = null;
            let y = null;

            if (fixedNodePositions[source]) {
              fixed = true;
              x = fixedNodePositions[source].x;
              y = fixedNodePositions[source].y;
            }

            nodes.push({
              name: source,
              category,
              symbolSize: size,
              itemStyle: { color },
              fixed,  // 固定属性
              x,      // x 坐标
              y       // y 坐标
            });
          }
          
          // 处理target节点
          if (!nodes.some(node => node.name === target)) {
            let size = 60;
            let color = customColors[target] || '#cccccc';
            let category = 'other';

            if (stigmaNodes.has(target)) {
              size = 60;
              category = 'stigma';
            } else if (noStigmaNodes.has(target)) {
              size = 60;
              category = 'no stigma';
            }

            // 检查是否需要固定位置
            let fixed = false;
            let x = null;
            let y = null;

            if (fixedNodePositions[target]) {
              fixed = true;
              x = fixedNodePositions[target].x;
              y = fixedNodePositions[target].y;
            }

            nodes.push({
              name: target,
              category,
              symbolSize: size,
              itemStyle: { color },
              fixed,  // 固定属性
              x,      // x 坐标
              y       // y 坐标
            });
          }

          // 处理 thirdElement 节点
          instance.triples.forEach(triple => {
            let [firstElement, , thirdElement] = triple;
            if (!thirdElement) return;  // 检查 thirdElement 是否存在

            // 如果 firstElement 和 target 相同，则处理边和节点
            if (firstElement === target) {
              let edgeKey = `${thirdElement}-${target}`;

              // 检查是否已经处理过此边
              if (!processedEdges.has(edgeKey)) {
                edges.push({
                  source: thirdElement,
                  target: target,
                  lineStyle: { color: customColors[target] || '#cccccc' },
                  label: 'related to'
                });
                processedEdges.add(edgeKey);
              }

              // 处理 thirdElement 的颜色，依据 ontology 中的类型
              let colorForThirdElement = '#cccccc'; // 默认颜色
              instance.ontology.forEach(ontology => {
                if (ontology[3] === thirdElement) {
                  let type = ontology[4]; // 获取类型
                  if (typeToColorMap[type]) {
                    colorForThirdElement = typeToColorMap[type]; // 使用类型对应的颜色
                  }
                  // 确保将类型赋值给 instance
                  if (!instance.type) {
                    instance.type = type;  // 将类型附加到 instance 中
                    console.log('Assigned type:', instance.type);  // 调试输出
                  }
                }
              });

              // 添加 thirdElement 节点，如果它尚不存在
              if (!nodes.some(node => node.name === thirdElement)) {
                nodes.push({
                  name: thirdElement,
                  category: 'other',
                  symbolSize: 10,
                  itemStyle: { color: colorForThirdElement },
                  label: { show: false }
                });
              }

              // 将 thirdElement 映射到 thirdElementToInstances 中
              if (!this.thirdElementToInstances[thirdElement]) {
                this.thirdElementToInstances[thirdElement] = [];
              }
              this.thirdElementToInstances[thirdElement].push(instance); // 添加实例
            }
          });
        });

        var option = {
          title: {
            // text: 'Stigma and Responsibility Network'
          },
          tooltip: {
            trigger: 'item',
            formatter: function (param) {
              if (param.dataType === 'edge') {
                return `${param.data.source} ${param.data.label} ${param.data.target}`;
              }
              return param.data.name;
            }
          },
          legend: {
            data: ['stigma', 'no stigma', 'positive cognitive judgment', 'negative cognitive judgment', 'positive feeling', 'negative feeling', 'positive behavior', 'negative behavior']
          },
          series: [
            {
              type: 'graph',
              layout: 'force',
              roam: true,
              label: {
                show: true,
                position: 'right',
                formatter: '{b}',
                emphasis: {
                  show: true
                }
              },
              draggable: false,
              categories: [
                { name: 'stigma', itemStyle: { color: '#ff6666' } },
                { name: 'no stigma', itemStyle: { color: '#66cc66' } },
                { name: 'positive cognitive judgment', itemStyle: { color: '#D98880' } },
                { name: 'negative cognitive judgment', itemStyle: { color: '#F5B7B1' } },
                { name: 'positive feeling', itemStyle: { color: '#BB8FCE' } },
                { name: 'negative feeling', itemStyle: { color: '#D7BDE2' } },
                { name: 'positive behavior', itemStyle: { color: '#87CEEB' } },
                { name: 'negative behavior', itemStyle: { color: '#ADD8E6' } },
                { name: 'other', itemStyle: { color: '#cccccc' } }
              ],
              data: nodes,
              links: edges,
              force: {
                edgeLength: [150, 300],
                repulsion: 1000,
                gravity: 0.1
              },
              emphasis: {
                focus: 'adjacency',
                lineStyle: {
                  width: 5
                }
              },
              edgeSymbol: ['circle', 'arrow'],
              edgeSymbolSize: [4, 10],
              lineStyle: {
                curveness: 0.3
              }
            }
          ]
        };

        myChart.setOption(option);

        // 监听图表的点击事件
        myChart.on('click', (params) => {
          const instances = this.thirdElementToInstances?.[params.data.name];
          
          if (instances && instances.length > 0) {
            this.selectedInstances = instances;
            this.currentIndex = 0; // 每次点击时重置到第一个实例
            console.log('Instances after chart click:', this.selectedInstances);
          }
        });
      })
      .catch(error => {
        console.error('Error fetching data:', error);
      });
  },
  methods: {
    // 展示前一个实例
    prevInstance() {
      if (this.currentIndex > 0) {
        this.currentIndex--;
      }
    },
    // 展示下一个实例
    nextInstance() {
      if (this.currentIndex < this.selectedInstances.length - 1) {
        this.currentIndex++;
        console.log('Next instance index:', this.currentIndex); // 调试输出
      }
    }
  }
};
</script>

<style scoped>
.container {
  display: flex;
  flex-direction: row;
  height: 100vh;
  overflow: hidden;
}

.chart {
  flex-grow: 1;
  min-width: 0;
  position: relative;
}

.filter {
  position: absolute;
  top: 10px;
  left: 10px; /* 放在右上角 */
  z-index: 1000;
  background-color: white;
  padding: 10px;
  border-radius: 5px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
}

.instance-content {
  width: 300px; /* 固定宽度 */
  padding: 20px;
  background-color: #f0f0f0;
  border-left: 1px solid #ccc;
  overflow-y: auto;
}

.instance-item {
  margin-bottom: 20px;
  text-align: left; /* 确保文本左对齐 */
  padding: 10px; /* 增加一些内边距，使文本不紧贴边框 */
}

.pagination {
  display: flex;
  justify-content: space-between;
  margin-top: 20px;
}

button {
  padding: 10px;
  font-size: 14px;
  cursor: pointer;
  position: relative;
}

button:disabled {
  cursor: not-allowed;
  opacity: 0.5;
}

h3 {
  margin-top: 0;
}

p {
  margin: 10px 0;
}
</style>