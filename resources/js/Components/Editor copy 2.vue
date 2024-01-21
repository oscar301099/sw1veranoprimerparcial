<template>
  <ul class="float-tool-bar">
    <li class="tool-btn" @click="_zoomIn">
      <Add weight="4" size="16" radius="100" color="#315B96"></Add>
    </li>
    <li class="tool-btn" @click="_zoomOut">
      <Delete height="4" width="16" radius="999" color="#315B96"></Delete>
    </li>
    <li class="tool-btn" @click="_undoModel"><img class="tool-btn-img-1" src="@/Assets/undo.png" /></li>
    <li class="tool-btn" @click="_redoModel"><img class="tool-btn-img-2" src="@/Assets/redo.png" /></li>
    <li class="tool-btn" @click="_clearCanvas"><img class="tool-btn-img-4" src="@/Assets/clean_icon.png" /></li>
  </ul>
  <div class="gojs-container flowchart-container" style="width: 100%; display: flex; justify-content: space-between">
    <div id="myPaletteDiv"
      style="width: 300px; margin-right: 10px; box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.1); background: rgba(0,0,0,.03);">
    </div>
    <div id=" goDiagramDiv"
      style="flex-grow: 1; height: 750px; border: solid 1px rgba(43, 52, 65, 0.4); background: rgba(0,0,0,.05);"></div>
  </div>
</template>

<script>
import go from 'gojs'
import Add from "@/Components/BasicComponents/Add.vue";
import Delete from "@/Components/BasicComponents/Delete.vue";
import {
  MessageLink,
  MessageDraggingTool,
  MessagingTool
} from './sequence'



// some parameters
var LinePrefix = 20 // vertical starting point in document for all Messages and Activations
var LineSuffix = 50 // vertical length beyond the last message time
var MessageSpacing = 20 // vertical distance between Messages at different steps
var ActivityWidth = 10 // width of each vertical activity bar
var ActivityStart = 5 // height before start message time
var ActivityEnd = 5 // height beyond end message time
var fill = 'green'
var lineV = {
  figure: 'LineV',
  fill: null,
  stroke: 'gray',
  strokeDashArray: [3, 3],
  width: 20,
  alignment: go.Spot.Center,
  portId: '',
  fromLinkable: true,
  fromLinkableDuplicates: true,
  fromLinkableSelfNode: true,
  toLinkable: true,
  toLinkableDuplicates: true,
  toLinkableSelfNode: true,
  cursor: 'pointer',
};


export default {
  name: 'SquenceDiagram',
  props: {
    diagram: {
      type: Object,
      default: () => { }
    },
  },
  data() {
    return {
      goDiagram: {},
      modelData: {
        class: 'go.GraphLinksModel'
      }
    }
  },
  watch: {
    /*  modelData: {
       handler: function (val) {
         const data = JSON.parse(JSON.stringify(val))
         Object.assign(this.modelData, data)
         if (this. goDiagram) {
           this.load()
         }
 
         console.log('watch')
       },
       deep: true,
     }, */
  },

  components: {
    Add,
    Delete,
  },
  async mounted() {
    await this.getDiagramContent();
    this.init()
  },
  methods: {
    init() {
      const _this = this
      MessagingTool.prototype.insertLink = function (fromnode, fromport, tonode, toport) {
        var newlink = go.LinkingTool.prototype.insertLink.call(this, fromnode, fromport, tonode, toport)

        if (newlink !== null) {
          var model = this.diagram.model
          // specify the time of the message
          var start = this.temporaryLink.time
          var duration = 1
          newlink.data.time = start
          model.setDataProperty(newlink.data, 'text', 'msg')
          // and create a new Activity node data in the "to" group data
          var newact = {
            group: newlink.data.to,
            start: start,
            duration: duration
          }
          model.addNodeData(newact)
          // now make sure all Lifelines are long enough
          _this.ensureLifelineHeights();
        }
        return newlink
      }

      let $ = go.GraphObject.make
      let diagram = $(go.Diagram, ' goDiagramDiv', {
        allowCopy: false,
        allowDrop: true,
        initialPosition: new go.Point(0, -150),
        initialContentAlignment: go.Spot.Center,

        linkingTool: $(MessagingTool), // defined below
        'resizingTool.isGridSnapEnabled': true,
        draggingTool: $(MessageDraggingTool), // defined below
        'draggingTool.gridSnapCellSize': new go.Size(1, MessageSpacing / 4),
        'draggingTool.isGridSnapEnabled': true,
        // automatically extend Lifelines as Activities are moved or resized
        SelectionMoved: this.ensureLifelineHeights,
        PartResized: this.ensureLifelineHeights,
        scrollsPageOnFocus: true,
        'undoManager.isEnabled': true,
      })

      diagram.grid =
        $(go.Panel, "Grid",
          $(go.Shape, "LineH", { strokeWidth: 0.5, strokeDashArray: [0, 9.5, 0.5, 0] })
        );
      //LifeLine
      diagram.groupTemplateMap.add('componentLifeLine',
        $(go.Group, 'Vertical',
          {
            locationSpot: go.Spot.Bottom,
            locationObjectName: 'HEADER',
            minLocation: new go.Point(-Infinity, 0),
            maxLocation: new go.Point(Infinity, 0),
            selectionObjectName: 'HEADER',
            doubleClick: function (e, node) {
              // Habilita la edición de texto al hacer doble clic en el encabezado
              var textBlock = node.findObject('TEXTBLOCK');
              if (textBlock) {
                diagram.commandHandler.editTextBlock(textBlock);
              }
            }
          },
          new go.Binding('location', 'loc', go.Point.parse).makeTwoWay(go.Point.stringify),
          $(go.Panel, 'Auto',
            { name: 'HEADER' },
            $(go.Shape, 'Rectangle',
              {
                fill: 'white',
                stroke: 'black',
                strokeWidth: 1.5,
                width: 80,
                height: 60
              }),
            $(go.TextBlock,
              {
                margin: 0,
                font: '400 10pt Source Sans Pro, sans-serif',
                stroke: 'black',
                editable: true,
                textEdited: function (textBlock, oldText, newText) {
                  var groupData = textBlock.part.data;
                  diagram.model.setDataProperty(groupData, 'text', newText);
                }
              },
              new go.Binding('text', 'text')),
          ),

          $(go.Shape,
            lineV,
            new go.Binding('height', 'duration', (duration) => {
              return LinePrefix + duration * MessageSpacing + LineSuffix
            }))
        )
      )

      diagram.groupTemplateMap.add('actorLifeLine',
        $(go.Group, 'Vertical',
          {
            locationSpot: go.Spot.Bottom,
            locationObjectName: 'HEADER',
            minLocation: new go.Point(-Infinity, 0),
            maxLocation: new go.Point(Infinity, 0),
            selectionObjectName: 'HEADER',
            doubleClick: function (e, node) {
              // Habilita la edición de texto al hacer doble clic en el encabezado
              var textBlock = node.findObject('TEXTBLOCK');
              if (textBlock) {
                diagram.commandHandler.editTextBlock(textBlock);
              }
            }
          },
          new go.Binding('location', 'loc', go.Point.parse).makeTwoWay(go.Point.stringify),
          $(go.Panel, 'Auto',
            { name: 'HEADER' },
            $(go.Picture,
              { source: '../images/actor.png', width: 30, height: 55 }
            ),
          ), $(go.TextBlock,
            {
              font: '400 10pt Source Sans Pro, sans-serif',
              stroke: 'black',
              editable: true,
              textEdited: function (textBlock, oldText, newText) {
                var groupData = textBlock.part.data;
                diagram.model.setDataProperty(groupData, 'text', newText);
              }
            },
            new go.Binding('text', 'text')),
          $(go.Shape, lineV,
            new go.Binding('height', 'duration', this.computeLifelineHeight))
        )
      )

      diagram.groupTemplateMap.add('entityLifeLine',
        $(go.Group, 'Vertical',
          {
            locationSpot: go.Spot.Bottom,
            locationObjectName: 'HEADER',
            minLocation: new go.Point(-Infinity, 0),
            maxLocation: new go.Point(Infinity, 0),
            selectionObjectName: 'HEADER',
            doubleClick: function (e, node) {
              // Habilita la edición de texto al hacer doble clic en el encabezado
              var textBlock = node.findObject('TEXTBLOCK');
              if (textBlock) {
                diagram.commandHandler.editTextBlock(textBlock);
              }
            }
          },
          new go.Binding('location', 'loc', go.Point.parse).makeTwoWay(go.Point.stringify),
          $(go.Panel, 'Auto',
            { name: 'HEADER' },
            $(go.Picture,
              { source: '../images/entity.png', width: 65, height: 65 }
            ),
          ), $(go.TextBlock,
            {
              font: '400 10pt Source Sans Pro, sans-serif',
              stroke: 'black',
              editable: true,
              textEdited: function (textBlock, oldText, newText) {
                var groupData = textBlock.part.data;
                diagram.model.setDataProperty(groupData, 'text', newText);
              }
            },
            new go.Binding('text', 'text')),
          $(go.Shape,
            lineV,
            new go.Binding('height', 'duration', (duration) => {
              return LinePrefix + duration * MessageSpacing + LineSuffix - 10
            }))
        )
      )

      diagram.groupTemplateMap.add('boundaryLifeLine',
        $(go.Group, 'Vertical',
          {
            locationSpot: go.Spot.Bottom,
            locationObjectName: 'HEADER',
            minLocation: new go.Point(-Infinity, 0),
            maxLocation: new go.Point(Infinity, 0),
            selectionObjectName: 'HEADER',
            doubleClick: function (e, node) {
              // Habilita la edición de texto al hacer doble clic en el encabezado
              var textBlock = node.findObject('TEXTBLOCK');
              if (textBlock) {
                diagram.commandHandler.editTextBlock(textBlock);
              }
            }
          },
          new go.Binding('location', 'loc', go.Point.parse).makeTwoWay(go.Point.stringify),
          $(go.Panel, 'Auto',
            { name: 'HEADER' },
            $(go.Picture,
              {
                source: '../images/boundary.png',
                width: 77,
                height: 65,
                margin: new go.Margin(0, 15, 0, 0)
              }
            ),
          ), $(go.TextBlock,
            {
              font: '400 10pt Source Sans Pro, sans-serif',
              stroke: 'black',
              editable: true,
              textEdited: function (textBlock, oldText, newText) {
                var groupData = textBlock.part.data;
                diagram.model.setDataProperty(groupData, 'text', newText);
              }
            },
            new go.Binding('text', 'text')),
          $(go.Shape,
            lineV,
            new go.Binding('height', 'duration', (duration) => {
              return LinePrefix + duration * MessageSpacing + LineSuffix - 10
            }))
        )
      )

      diagram.groupTemplateMap.add('controlLifeLine',
        $(go.Group, 'Vertical',
          {
            locationSpot: go.Spot.Bottom,
            locationObjectName: 'HEADER',
            minLocation: new go.Point(-Infinity, 0),
            maxLocation: new go.Point(Infinity, 0),
            selectionObjectName: 'HEADER',
            doubleClick: function (e, node) {
              // Habilita la edición de texto al hacer doble clic en el encabezado
              var textBlock = node.findObject('TEXTBLOCK');
              if (textBlock) {
                diagram.commandHandler.editTextBlock(textBlock);
              }
            }
          },
          new go.Binding('location', 'loc', go.Point.parse).makeTwoWay(go.Point.stringify),
          $(go.Panel, 'Auto',
            { name: 'HEADER' },
            $(go.Picture,
              {
                source: '../images/control.png',
                width: 65,
                height: 75,

              }
            ),
          ), $(go.TextBlock,
            {
              font: '400 10pt Source Sans Pro, sans-serif',
              stroke: 'black',
              editable: true,
              textEdited: function (textBlock, oldText, newText) {
                var groupData = textBlock.part.data;
                diagram.model.setDataProperty(groupData, 'text', newText);
              }
            },
            new go.Binding('text', 'text')),
          $(go.Shape,
            lineV,
            new go.Binding('height', 'duration', (duration) => {
              return LinePrefix + duration * MessageSpacing + LineSuffix - 10
            }))
        )
      )

      //Activity node
      diagram.nodeTemplate = $(go.Node,
        {
          locationSpot: go.Spot.Top,
          locationObjectName: 'SHAPE',
          minLocation: new go.Point(NaN, LinePrefix - ActivityStart),
          maxLocation: new go.Point(NaN, 19999),
          selectionObjectName: 'SHAPE',
          resizable: true,
          resizeObjectName: 'SHAPE',
          resizeAdornmentTemplate:
            $(go.Adornment, 'Spot',
              $(go.Placeholder),
              $(go.Shape, // only a bottom resize handle
                {
                  alignment: go.Spot.Bottom,
                  cursor: 'col-resize',
                  desiredSize: new go.Size(6, 6),
                  fill: 'yellow'
                })
            )
        },
        new go.Binding('location', '', this.computeActivityLocation).makeTwoWay(this.backComputeActivityLocation),
        $(go.Shape, 'Rectangle',
          {
            name: 'SHAPE',
            fill: 'white',
            stroke: 'black',
            width: ActivityWidth,
            // allow Activities to be resized down to 1/4 of a time unit
            minSize: new go.Size(ActivityWidth, this.computeActivityHeight(0.25))
          },
          new go.Binding('height', 'duration', this.computeActivityHeight).makeTwoWay(this.backComputeActivityHeight))
      )

      // define the Message Link template.
      diagram.linkTemplate = $(MessageLink, // defined below
        { selectionAdorned: true, curviness: 0 },
        $(go.Shape, 'Rectangle',
          { stroke: 'black' }),
        $(go.Shape,
          { toArrow: 'OpenTriangle', stroke: 'black' }),
        $(go.TextBlock,
          {
            font: '400 9pt Source Sans Pro, sans-serif',
            segmentIndex: 0,
            segmentOffset: new go.Point(NaN, NaN),
            isMultiline: false,
            editable: true
          },
          new go.Binding('text', 'text').makeTwoWay())
      )

      diagram.groupTemplateMap.add("fragment",
        $(go.Group, "Auto",
          {
            resizable: true,  // Permite redimensionar el grupo
            resizeObjectName: "SHAPE",  // El objeto que se redimensiona es el rectángulo principal
          },
          $(go.Shape, "Rectangle",  // Rectángulo principal
            { name: "SHAPE", fill: "transparent", width: 250, height: 150 },
            new go.Binding("width", "width").makeTwoWay(),
            new go.Binding("height", "height").makeTwoWay()),
          $(go.TextBlock,  // la etiqueta del elemento opt
            {
              alignment: go.Spot.TopLeft,
              font: "bold 11pt sans-serif",
              margin: new go.Margin(5, 0, 5, 10),
              editable: false,
            },
            new go.Binding("text", "text")),
          $(go.Panel, "Auto",

            $(go.Shape, "Rectangle",  // Rectángulo secundario
              {
                fill: 'transparent',
                stroke: 'black',
                margin: new go.Margin(35, 0, 0, 0)
              },
              new go.Binding("width", "", function (group) {
                return group.actualBounds.width;
              }).ofObject("SHAPE"),
              new go.Binding("height", "", function (group) {
                return group.actualBounds.height - 10;
              }).ofObject("SHAPE"))
          )
        ))

      // Plantilla para el elemento "Alt"
      diagram.groupTemplateMap.add('alt',
        $(go.Group, 'Vertical',
          { background: 'lightgreen', resizable: false, selectable: true },
          $(go.TextBlock, 'Alt', { margin: 5, font: 'bold 12px sans-serif' }),
          $(go.Panel, 'Horizontal',
            { padding: 5 },
            $(go.Shape, 'Rectangle', { fill: 'lightgreen', width: 200, height: 200 }),
            $(go.TextBlock, 'Alternative 1', { font: '12px sans-serif' })
          )
        )
      );

      // Plantilla para el elemento "Loop"
      diagram.groupTemplateMap.add('loop',
        $(go.Group, 'Vertical',
          { background: 'lightcoral', resizable: false, selectable: false },
          $(go.TextBlock, 'Loop', { margin: 5, font: 'bold 12px sans-serif' }),
          $(go.Panel, 'Horizontal',
            { padding: 5 },
            $(go.Shape, 'Circle', { fill: 'lightcoral', width: 20, height: 20 }),
            $(go.TextBlock, 'Loop 1', { font: '12px sans-serif' })
          )
        )
      );

      const myPalette = $(
        go.Palette,
        'myPaletteDiv',
        {
          scrollsPageOnFocus: false,
          nodeTemplateMap: diagram.nodeTemplateMap,
          groupTemplateMap: diagram.groupTemplateMap,
          initialScale: 0.8,
          layout: $(go.GridLayout,
            { wrappingColumn: 3, alignment: go.GridLayout.Position, cellSize: new go.Size(50, 50) })
        }
      )

      myPalette.model = new go.GraphLinksModel([
        { category: 'actorLifeLine', key: 'actorLifeLine', text: 'Actor', isGroup: true, duration: 5 },
        { category: 'componentLifeLine', key: 'componentLifeLine', text: 'Object', isGroup: true, duration: 5 },
        { category: 'entityLifeLine', key: 'entityLifeLine', text: 'Entity', isGroup: true, duration: 5 },
        { category: 'boundaryLifeLine', key: 'boundaryLifeLine', text: 'boundary', isGroup: true, duration: 5 },
        { category: 'controlLifeLine', key: 'controlLifeLine', text: 'control', isGroup: true, duration: 5 },
        { category: 'fragment', key: 'alt', text: 'alt', isGroup: true, },
        { category: 'fragment', key: 'opt', text: 'opt', isGroup: true, },
        { category: 'fragment', key: 'loop', text: 'loop', isGroup: true },
      ])


      console.log('----------------')
      console.log(this.modelData)
      diagram.model = go.Model.fromJson(this.modelData)

      diagram.grid.visible = true;
      this.goDiagram = diagram;

      this.goDiagram.addDiagramListener('ExternalObjectsDropped', function (evt) {
        window.setTimeout(function () {
          _this.distributionId()
        }, 100);
      })

      this.goDiagram.addModelChangedListener(e => { if (e.isTransactionFinished) { this.modelData = e.model; this.updateDiagramContent() } })
    },

    _undoModel() {
      this.goDiagram.undoManager.undo();
    },

    distributionId() {
      this.saveSecChart()
      this.modelData = JSON.parse(this.modelData)
      for (const item of this.modelData.nodeDataArray) {
        if (item.key < 0) {
          item.key = this.newGuid()
        }
      }
    },

    saveSecChart() {
      console.log('savesechart()')
      this.modelData = this.goDiagram.model.toJson();
    },

    newGuid() {
      var guid = ''
      for (var i = 1; i <= 32; i++) {
        var n = Math.floor(Math.random() * 16.0).toString(16)
        guid += n
        if (i === 8 || i === 12 || i === 16 || i === 20) guid += '-'
      }
      return guid
    },

    loadFlowChart() {
      this.goDiagram.model = go.Model.fromJson(this.modelData)
      this.goDiagram.animationManager.stopAnimation()
    },

    computeLifelineHeight(duration) {
      return LinePrefix + duration * MessageSpacing + LineSuffix;
    },

    computeActivityLocation(act) {
      var groupdata = this.goDiagram.model.findNodeDataForKey(act.group)
      if (groupdata === null) return new go.Point()
      // get location of Lifeline's starting point
      var grouploc = go.Point.parse(groupdata.loc)
      return new go.Point(grouploc.x, this.convertTimeToY(act.start) - ActivityStart)
    },
    // time is just an abstract small non-negative integer
    // here we map between an abstract time and a vertical position
    convertTimeToY(t) {
      return t * MessageSpacing + LinePrefix
    },

    convertYToTime(y) {
      return (y - LinePrefix) / MessageSpacing
    },

    backComputeActivityLocation(loc, act) {
      this.goDiagram.model.setDataProperty(act, 'start', this.convertYToTime(loc.y + ActivityStart))
    },

    computeActivityHeight(duration) {
      return ActivityStart + duration * MessageSpacing + ActivityEnd
    },

    backComputeActivityHeight(height) {
      return (height - ActivityStart - ActivityEnd) / MessageSpacing
    },

    ensureLifelineHeights(e) {
      // iterate over all Activities (ignore Groups)
      var arr = this.goDiagram.model.nodeDataArray
      var max = -1
      for (let i = 0; i < arr.length; i++) {
        var act = arr[i]
        if (act.isGroup) continue // 跳过组
        max = Math.max(max, act.start + act.duration)
      }

      if (max > 0) {
        // now iterate over only Groups
        // eslint-disable-next-line no-redeclare
        for (let i = 0; i < arr.length; i++) {
          var gr = arr[i]
          if (!gr.isGroup) continue // 跳过非租
          if (max > gr.duration) { // this only extends, never shrinks
            this.goDiagram.model.setDataProperty(gr, 'duration', max)
          }
        }
      }
    },

    load() {
      // this.diagram.model = go.Model.fromJson(document.getElementById('mySavedModel').value)
      //this. goDiagram.model = go.Model.fromJson(this.modelData)
    },

    async getDiagramContent() {
      await axios.get(route('api.diagrams.get', this.diagram.id))
        .then((response) => {
          console.log('getdiagram------------------')
          //console.log(response.data.content)
          this.modelData = response.data.content
        })
        .catch(error => console.log(error))
    },

    updateDiagramContent() {
      //console.log('updateDiagramContent()')
      axios.put(route('api.diagrams.update', this.diagram.id),
        {
          content: this.modelData.toJson()
        })
        .then((response) => {
          console.log(response.data)
        })
        .catch(error => console.log(error))
    },

    _clearCanvas() {
      this.goDiagram.clear();
    }

  }
}
</script>


<style>
@import '@/Assets/styles/secuence-diagram.css';
</style>
