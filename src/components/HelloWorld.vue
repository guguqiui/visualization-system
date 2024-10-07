<template>
  <div class="container">
    <!-- 图表部分 -->
    <div id="chart" class="chart"></div>
    
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
      selectedInstances: [], // Store the currently selected instance list
      currentIndex: 0, // Current instance index for display
      thirdElementToInstances: {}, // Mapping from thirdElement to multiple instances
      edgeVisibility: {}, // Track edge visibility for stigmaNodes and noStigmaNodes
      edges: [], // Store edges for toggling
      nodes: [],
      myChart: null // Reference to the chart instance
    };
  },
  mounted() {
    var chartDom = document.getElementById('chart');
    this.myChart = echarts.init(chartDom); // Assign to this.myChart

    // Define color mappings for different ontology types (typeToColorMap)
    const typeToColorMap = {
      'cognitive judgment': '#FF6666',
      'signaling event': '#66CC66',
      'nature': '#FFCC66',
      'suggestion': '#66CCCC',
      'belief': '#9966CC',
      'feeling': '#FF99CC',
      'situation': '#FF9966',
      'past experience': '#6699FF',
      'potential outcome': '#FFCC99',
      'behavior': '#CC66FF',
      'intention': '#FF6699'
    };

    // attribution colors (customColors for attribution types)
    const customColors = {
      'stigma':'#ff6666',
      'no stigma': '#66cc66',
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

    // Edge visibility for all stigma and no stigma nodes
    stigmaNodes.forEach(node => this.edgeVisibility[node] = false);
    noStigmaNodes.forEach(node => this.edgeVisibility[node] = false);

    // Fetch the data and process nodes/edges
    fetch('/data/test.json')
      .then(response => response.json())
      .then(data => {

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

        // Create nodes and hide initial edges
        data.forEach(instance => {
          let source = instance.category_;  // Category (source node)
          let target = instance.attribution_type_;  // Attribution (target node)

          // Check if the source node has a fixed position
          let fixed = false, x = null, y = null;
          if (fixedNodePositions[source]) {
            fixed = true;
            x = fixedNodePositions[source].x;
            y = fixedNodePositions[source].y;
          }

          // Process the source node
          if (!this.nodes.some(node => node.name === source)) {
            // Assign color based on the ontology type (using typeToColorMap)
            let nodeColor = customColors[source] || '#cccccc';

            let size = 100;  // Default size for the source node
            let category = stigmaNodes.has(source) ? 'stigma' : noStigmaNodes.has(source) ? 'no stigma' : 'other';

            // Add the source node to the list if it doesn't exist
            this.nodes.push({
              name: source,
              category,
              symbolSize: size,
              itemStyle: { color: nodeColor },
              fixed,  // If the node is fixed, mark it
              x,      // x position if fixed
              y       // y position if fixed
            });
          }

          // Process the target node (attribution_type_)
          if (!this.nodes.some(node => node.name === target)) {
            let nodeColor = customColors[target] || '#cccccc';  // Assign a color to the target node

            let size = 60;  // Default size for the target node
            let category = stigmaNodes.has(target) ? 'stigma' : noStigmaNodes.has(target) ? 'no stigma' : 'other';

            // Check for fixed positions for the target node
            fixed = false;
            x = null;
            y = null;
            if (fixedNodePositions[target]) {
              fixed = true;
              x = fixedNodePositions[target].x;
              y = fixedNodePositions[target].y;
            }

            // Add the target node if it doesn't already exist
            this.nodes.push({
              name: target,
              category,
              symbolSize: size,
              itemStyle: { color: nodeColor },
              fixed,  // If the node is fixed, mark it
              x,      // x position if fixed
              y       // y position if fixed
            });
          }

          // Create the edge between the source and target
          let edge = {
            source: target,  // Target first in this context
            target: source,  // Source second
            label: 'related to',  // Edge label
            lineStyle: { color: customColors[source] || '#cccccc' },  // Edge color based on source node
            // invisible: true  // Initially hide the edge
          };

          // Add the edge to the list
          this.edges.push(edge);

          // 处理 thirdElement 节点
          instance.triples.forEach(triple => {
            let [firstElement, , thirdElement] = triple;
            if (!thirdElement) return;  // 检查 thirdElement 是否存在

            // 如果 firstElement 和 target 相同，则处理边和节点
            if (firstElement === target) {
              let edgeKey = `${thirdElement}-${target}`;

              // 检查是否已经处理过此边
              if (!processedEdges.has(edgeKey)) {
                this.edges.push({
                  source: thirdElement,
                  target: target,
                  lineStyle: { 
                    color: customColors[target] || '#cccccc',
                    opacity: 0 // 初始不可见
                   },
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
                  }
                }
              });

              // 添加 thirdElement 节点，如果它尚不存在
              if (!this.nodes.some(node => node.name === thirdElement)) {
                this.nodes.push({
                  name: thirdElement,
                  category: 'other',
                  symbolSize: 10,
                  itemStyle: { 
                    color: colorForThirdElement,
                    opacity: 0
                   },
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
            data: ['stigma', 'no stigma', 'cognitive judgment', 'feeling', 'behavior']
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
                { name: 'cognitive judgment', itemStyle: { color: '#F5B7B1' } },
                { name: 'feeling', itemStyle: { color: '#D7BDE2' } },
                { name: 'behavior', itemStyle: { color: '#ADD8E6' } },
                { name: 'other', itemStyle: { color: '#cccccc' } }
              ],
              data: this.nodes,
              links: this.edges,
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

        this.myChart.setOption(option);

        // 监听图表的点击事件
        this.myChart.on('click', (params) => {
          console.log('Clicked node:', params.data.name);  
          const instances = this.thirdElementToInstances?.[params.data.name];
          
          if (instances && instances.length > 0) {
            this.selectedInstances = instances;
            this.currentIndex = 0; // 每次点击时重置到第一个实例
            // console.log('Instances after chart click:', this.selectedInstances);
          }
          
          // 增加对 stigmaNodes 和 noStigmaNodes 的检查并切换边的可见性
          if (stigmaNodes.has(params.data.name) || noStigmaNodes.has(params.data.name)) {
            console.log(params.data.name);
            this.toggleEdges(params.data.name);  // 调用切换边的函数
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
    },
    // Toggle edges for a specific node
    toggleEdges(nodeName) {
      this.edgeVisibility[nodeName] = !this.edgeVisibility[nodeName]; // 更新节点的可见性状态
      let sourceNodes = new Set(); // 找到所有指向 nodeName 的节点

      // 遍历所有边，找到与点击节点相关的边并切换可见性
      this.edges.forEach(edge => {
        if (edge.target === nodeName) {
          // 使用 lineStyle.opacity 来控制边的可见性
          edge.lineStyle = edge.lineStyle || {};
          edge.lineStyle.opacity = this.edgeVisibility[nodeName] ? 1 : 0;  // 切换可见性
          sourceNodes.add(edge.source); // 将源节点添加到 sourceNodes 集合中
        }
      });

      // 改变点的可见性
      sourceNodes.forEach(sourceNode => {
        this.nodes.forEach(node => {
          if (node.name === sourceNode) {
            // node.symbolSize = 10;  // 设置节点的大小为可见状态
            node.itemStyle.opacity = 1;  // 设置节点的透明度为可见状态
          }
        });
      });

      // 确保图表根据新的可见性状态更新
      this.myChart.setOption({
        series: [
          {
            links: this.edges, // 更新边的可见性
            data: this.nodes  // 更新节点
          }
        ]
      });

      console.log('Chart updated with new edge visibility for node:', nodeName);  // 确认图表更新
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