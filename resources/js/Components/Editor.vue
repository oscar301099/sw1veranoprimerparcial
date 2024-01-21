<template>
  <ul class="float-tool-bar">
    <li class="tool-btn" @click="zoomIn">
      <Add weight="4" size="16" radius="100" color="#315B96"></Add>
    </li>
    <li class="tool-btn" @click="zoomOut">
      <Delete height="4" width="16" radius="999" color="#315B96"></Delete>
    </li>
    <li class="tool-btn" @click="undo"><img class="tool-btn-img-1" src="@/Assets/undo.png" /></li>
    <li class="tool-btn" @click="redo"><img class="tool-btn-img-2" src="@/Assets/redo.png" /></li>
    <li class="tool-btn" @click="clearCanvas"><img class="tool-btn-img-4" src="@/Assets/clean_icon.png" /></li>
  </ul>

  <div class="grid text-sm">
    <div class="grid lg:grid-cols-5 sm:grid-cols-2 mb-4 -mt-4 justify-between">
      <div class="col-span-2">
        <span v-for="user in onlineUsers">
          <i class="fas fa-circle fa-sm fa-fw text-green-500"></i>{{ ' ' + user.name + ' ' + ' ' }}
        </span>
      </div>
      <div class="col-span-2">
        <div class="flex justify-end">
          <!-- <SecondaryButton @click="exportAsPNG" class="ml-1">PNG</SecondaryButton> -->
          <SecondaryButton @click="exportAsJPG" class="ml-1">JPEG</SecondaryButton>
          <SecondaryButton @click="exportAsJSON" class="ml-1">JSON</SecondaryButton>
          <SecondaryButton @click="exportAsXMI" class="ml-1">XMI</SecondaryButton>
        </div>
      </div>
      <div class="col-span-1" v-if="props.project.user_id == $page.props.auth.user.id">
        <div class="flex justify-end">
          <PrimaryButton @click="saveDiagramContent(false)" class="ml-5">GUARDAR</PrimaryButton>
        </div>
      </div>

    </div>

  </div>
  <div class="grid grid-cols-4 w-full">
    <div class="col col-span-1">
      <div class="row">
        <div id="myPaletteDiv" class="flex"
          style="width: 100%; height: 500px; margin-right: 10px; box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.1); background: rgba(0,0,0,.03)">
        </div>
      </div>
      <div class="row pt-5 pr-5">
        <ul class="grid w-full gap-2 md:grid-rows-2">
          <li>
            <input type="radio" id="arrow_triangle" name="synchronous" value="yes" v-model="synchronous"
              class="hidden peer" required>
            <label for="arrow_triangle"
              class="inline-flex items-center justify-center w-full py-2 text-gray-500 bg-white border border-gray-200 rounded-lg cursor-pointer peer-checked:border-purple-600 peer-checked:text-purple-600 hover:text-gray-600 hover:bg-gray-100 ">
              <img :src="'../images/arrow_triangle.png'" alt="open arrow" height="10" width="180">
            </label>
          </li>
          <li>
            <input type="radio" id="open_arrow_triangle" name="synchronous" value="no" v-model="synchronous"
              class="hidden peer">
            <label for="open_arrow_triangle"
              class="inline-flex items-center justify-center w-full py-2 text-gray-500 bg-white border border-gray-200 rounded-lg cursor-pointer peer-checked:border-purple-600 peer-checked:text-purple-600 hover:text-gray-600 hover:bg-gray-100 ">
              <img :src="'../images/arrow_open_triangle.png'" alt="open arrow" height="10" width="180">
            </label>
          </li>
        </ul>
      </div>
    </div>
    <div class="col col-span-3">
      <div id="goDiagramDiv"
        style="width: 100%; height: 670px; border: solid 1px rgba(43, 52, 65, 0.4); background: rgba(0,0,0,.05);">
      </div>
    </div>
  </div>
</template>

<script setup>
import go from 'gojs'
import Add from "@/Components/BasicComponents/Add.vue";
import Delete from "@/Components/BasicComponents/Delete.vue";
import SecondaryButton from '@/Components/SecondaryButton.vue';
import PrimaryButton from './PrimaryButton.vue';

import {
  ref,
  onMounted
} from 'vue';


const $ = go.GraphObject.make;
const synchronous = ref('yes')
let myDiagram;
let modelData;
let onlineUsers = ref([]);
const LinePrefix = 20;
const LineSuffix = 30;
const MessageSpacing = 20;
const ActivityWidth = 10;
const ActivityStart = 5;
const ActivityEnd = 5;

const props = defineProps({
  diagram: {
    type: Object, default: () => { }
  },
  project: { type: Object }
})

class MessageLink extends go.Link {
  constructor() {
    super();
    this.time = 0;
    this.direction = 0;
  }

  getLinkPoint(node, port, spot, from, ortho, othernode, otherport) {
    const p = port.getDocumentPoint(go.Spot.Center);
    const r = port.getDocumentBounds();
    const op = otherport.getDocumentPoint(go.Spot.Center);

    const data = this.data;
    const time = data !== null ? data.time : this.time;  // if not bound, assume this has its own "time" property

    const aw = this.findActivityWidth(node, time);
    const x = (op.x > p.x ? p.x + aw / 2 : p.x - aw / 2);
    const y = Math.max(convertTimeToY(time), 180);
    return new go.Point(x, y);
  }

  findActivityWidth(node, time) {
    let aw = ActivityWidth;
    if (node instanceof go.Group) {
      // see if there is an Activity Node at this point -- if not, connect the link directly with the Group's lifeline
      if (!node.memberParts.any(mem => {
        const act = mem.data;
        return (act !== null && act.start <= time && time <= act.start + act.duration);
      })) {
        aw = 0;
      }
    }
    return aw;
  }

  getLinkDirection(node, port, linkpoint, spot, from, ortho, othernode, otherport) {
    const p = port.getDocumentPoint(go.Spot.Center);
    const op = otherport.getDocumentPoint(go.Spot.Center);
    const right = op.x > p.x;
    return right ? 0 : 180;
  }

  computePoints() {
    if (this.fromNode === this.toNode) {  // also handle a reflexive link as a simple orthogonal loop
      const data = this.data;
      const time = data !== null ? data.time : this.time;  // if not bound, assume this has its own "time" property
      const p = this.fromNode.port.getDocumentPoint(go.Spot.Center);
      const aw = this.findActivityWidth(this.fromNode, time);

      const x = p.x + aw / 2;
      const y = convertTimeToY(time);
      this.clearPoints();
      this.addPoint(new go.Point(x, y));
      this.addPoint(new go.Point(x + 50, y));
      this.addPoint(new go.Point(x + 50, y + 5));
      this.addPoint(new go.Point(x, y + 5));
      return true;
    } else {
      return super.computePoints();
    }
  }
}

class MessagingTool extends go.LinkingTool {
  constructor() {
    super();

    const $ = go.GraphObject.make;

    this.temporaryLink =
      $(MessageLink,
        $(go.Shape, "Rectangle",
          { stroke: "magenta", strokeWidth: 2 }),
      );
  }

  doActivate() {
    super.doActivate();
    const time = convertYToTime(this.diagram.firstInput.documentPoint.y);
    this.temporaryLink.time = Math.ceil(time);  // round up to an integer value
  }

  insertLink(fromnode, fromport, tonode, toport) {
    const newlink = super.insertLink(fromnode, fromport, tonode, toport);
    
    if (newlink !== null) {
      const model = this.diagram.model;
      // specify the time of the message
      const start = this.temporaryLink.time;
      const duration = 1;
      newlink.data.time = start;

      model.setDataProperty(newlink.data, "text", "msg");
      model.setDataProperty(newlink.data, "synchronous", synchronous.value);
      // and create a new Activity node data in the "to" group data
      const newact = {
        group: newlink.data.to,
        start: start,
        duration: duration
      };

      model.addNodeData(newact);
      // now make sure all Lifelines are long enough
      ensureLifelineHeights();
    }
    return newlink;
  }
}

class MessageDraggingTool extends go.DraggingTool {
  computeEffectiveCollection(parts, options) {
    const result = super.computeEffectiveCollection(parts, options);
    // add a dummy Node so that the user can select only Links and move them all
    result.add(new go.Node(), new go.DraggingInfo(new go.Point()));
    parts.each(part => {
      if (part instanceof go.Link) {
        result.add(part, new go.DraggingInfo(part.getPoint(0).copy()));
      }
    })
    return result;
  }

  mayMove() {
    return !this.diagram.isReadOnly && this.diagram.allowMove;
  }

  moveParts(parts, offset, check) {
    super.moveParts(parts, offset, check);
    const it = parts.iterator;
    while (it.next()) {
      if (it.key instanceof go.Link) {
        const link = it.key;
        const startY = it.value.point.y;  // DraggingInfo.point.y
        //let y = startY + offset.y;  // determine new Y coordinate value for this link
        let y = Math.max(180, startY + offset.y);
        const cellY = this.gridSnapCellSize.height;
        y = Math.round(y / cellY) * cellY;  // snap to multiple of gridSnapCellSize.height
        const t = Math.max(0, convertYToTime(y));
        link.diagram.model.set(link.data, "time", t);
        link.invalidateRoute();
      }
    }
  }
}

onMounted(async () => {
  connectSocket();
  initEditor();
  initPalette();
})

const getDiagramContent = async () => {
  await axios.get(route('api.diagrams.get', props.diagram.id))
    .then((response) => {
      modelData = response.data.content
    })
    .catch(error => console.log(error))
}

const saveDiagramContent = async (inCache) => {
  if (modelData !== myDiagram.model.toJson() || !inCache) {
    modelData = myDiagram.model.toJson();
    updateDiagramContent(modelData, inCache);
    console.log('nueva modificacion')
    console.log(modelData);
  }
}

const updateDiagramContent = async (content, inCache) => {
  axios.put(route('api.diagrams.update', props.diagram.id),
    {
      content: content, cache: inCache
    })
    .then((response) => {
      console.log('updateDiagramContent')
    })
    .catch(error => console.log(error))
};

function computeLifelineHeight(duration) {
  return LinePrefix + duration * MessageSpacing + LineSuffix;
}

function computeActivityLocation(act) {
  const groupdata = myDiagram.model.findNodeDataForKey(act.group);
  if (groupdata === null) return new go.Point();
  const grouploc = go.Point.parse(groupdata.loc);
  return new go.Point(grouploc.x, convertTimeToY(act.start) - ActivityStart);
}

function backComputeActivityLocation(loc, act) {
  myDiagram.model.setDataProperty(act, "start", convertYToTime(loc.y + ActivityStart));
}

function computeActivityHeight(duration) {
  return ActivityStart + duration * MessageSpacing + ActivityEnd;
}

function backComputeActivityHeight(height) {
  return (height - ActivityStart - ActivityEnd) / MessageSpacing;
}

function convertTimeToY(t) {
  return t * MessageSpacing + LinePrefix;
}

function convertYToTime(y) {
  return (y - LinePrefix) / MessageSpacing;
}

function ensureLifelineHeights(e) {
  const arr = myDiagram.model.nodeDataArray;
  let max = -1;
  for (let i = 0; i < arr.length; i++) {
    const act = arr[i];
    if (act.isGroup) continue;
    max = Math.max(max, act.start + act.duration);
  }
  if (max > 0) {
    // now iterate over only Groups
    for (let i = 0; i < arr.length; i++) {
      const gr = arr[i];
      if (!gr.isGroup) continue;
      if (max > gr.duration) {  // this only extends, never shrinks
        myDiagram.model.setDataProperty(gr, "duration", max);
      }
    }
  }
}

const clearCanvas = () => {
  myDiagram.clear();
}

async function load() {
  await getDiagramContent();
  myDiagram.model = go.Model.fromJson(modelData)
}

function undo() {
  myDiagram.commandHandler.undo();
}

function redo() {
  myDiagram.commandHandler.redo();
}

function zoomIn() {
  myDiagram.commandHandler.increaseZoom();
  myDiagram.commandHandler.increaseZoom();
}

function zoomOut() {
  myDiagram.commandHandler.decreaseZoom();
  myDiagram.commandHandler.decreaseZoom();
}

function exportAsPNG() {
  var blob = myDiagram.makeImageData({
    returnType: "blob",
    scale: 3,
    callback: function (blob) {
      myCallback(blob, '.png');
    }
  });
}

function exportAsJPG() {
  var blob = myDiagram.makeImageData({
    background: "white",
    returnType: "blob",
    scale: 3,
    callback: function (blob) {
      myCallback(blob, '.jpg');
    }
  });
}

function myCallback(blob, extension) {
  var url = window.URL.createObjectURL(blob);
  var filename = 'diagram ' + props.diagram.name + extension;

  var a = document.createElement("a");
  a.style = "display: none";
  a.href = url;
  a.download = filename;

  if (window.navigator.msSaveBlob !== undefined) {
    window.navigator.msSaveBlob(blob, filename);
    return;
  }

  document.body.appendChild(a);
  requestAnimationFrame(() => {
    a.click();
    window.URL.revokeObjectURL(url);
    document.body.removeChild(a);
  });
}

const exportAsJSON = () => {
  const jsonData = myDiagram.model.toJson();

  const blob = new Blob([jsonData], { type: 'application/json' });
  const url = URL.createObjectURL(blob);
  const link = document.createElement('a');
  link.href = url;
  link.download = 'diagrama.json';
  link.click();
}

const exportAsCode = async (language) => {
  let routeExport = 'api.diagrams.exports';
  let extension = '';
  switch (language) {
    case 'java':
      routeExport = routeExport + '.java';
      extension = '.java';
      break;
    case 'php':
      routeExport = routeExport + '.php';
      extension = '.php';
      break;
    case 'csharp':
      routeExport = routeExport + '.csharp';
      extension = '.cs';
      break;
    default:
      break;
  };

  await axios.get(route(routeExport, props.diagram.id))
    .then((response) => {
      const blob = new Blob([response.data], { type: 'text/plain' });

      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');

      a.href = url;
      a.download = toPascalCase(props.diagram.name) + extension;

      a.click();

      URL.revokeObjectURL(url);
    })
    .catch(error => console.log(error)
    )
}

const toPascalCase = (text) => {
  const words = text.split(" ");
  const capitalized = words.map((word) => {
    return word.charAt(0).toUpperCase() + word.slice(1).toLowerCase();
  });
  return capitalized.join("");
}

const exportAsXMI = async () => {
  await axios.get(route('api.diagrams.exports.xmi', props.diagram.id))
    .then((response) => {
      const texto = response.data;
      const blob = new Blob([texto], { type: 'application/xmi+xml' });
      const url = URL.createObjectURL(blob);

      const enlace = document.createElement('a');
      enlace.download = 'archivo.xmi';
      enlace.href = url;

      enlace.click();
      URL.revokeObjectURL(url);
    })
    .catch(error => console.log(error))

}

const connectSocket = () => {
  window.Echo
    .join(`room.${props.diagram.id}`)
    .here(users => {
      console.log('entraste')
      onlineUsers.value = users;

      console.log(onlineUsers.value)
      //this.systemMessage('You have joined the channel.');
    })
    .joining(user => {
      console.log('alguien entró')
      onlineUsers.value.push(user);
      console.log(onlineUsers.value)
      //this.systemMessage(`${user.name} joined the channel.`);
    })
    .leaving(user => {
      console.log('alguien salió')
      onlineUsers.value.splice(onlineUsers.value.indexOf(user), 1);
      //this.systemMessage(`${user.name} left the channel.`);
    })
    .error((error) => {
      console.log(JSON.stringify(error))
      //this.errorMessage(`Received error: ${JSON.stringify(error)}`);
    })
    .listen('.room.content', ({ content }) => {
      if (content.toString() != modelData.toString()) {
        console.log('modelo actualizado por evento')
        console.log('---------------')
        myDiagram.animationManager.isEnabled = false;
        modelData = content;
        myDiagram.model = go.Model.fromJson(content)
        myDiagram.animationManager.isEnabled = true;
      }
      /* this.userMessage(message, user); */
    });
}

const lineV = {
  figure: 'LineV',
  fill: null,
  stroke: 'gray',
  strokeDashArray: [3, 3],
  width: 20,
  alignment: go.Spot.Center,
  portId: '',
  fromLinkable: true,
  fromLinkableDuplicates: true,
  toLinkable: true,
  toLinkableDuplicates: true,
  cursor: 'pointer',
};

const initEditor = async () => {
  myDiagram = new go.Diagram("goDiagramDiv", {
    allowCopy: false,
    linkingTool: $(MessagingTool),
    "resizingTool.isGridSnapEnabled": true,
    draggingTool: $(MessageDraggingTool),
    "draggingTool.gridSnapCellSize": new go.Size(1, MessageSpacing / 4),
    "draggingTool.isGridSnapEnabled": true,
    "SelectionMoved": ensureLifelineHeights,
    "PartResized": ensureLifelineHeights,
    "undoManager.isEnabled": true,
    initialAutoScale: go.Diagram.Uniform,
    initialContentAlignment: go.Spot.Center,
    initialPosition: new go.Point(500, -200),
    scrollsPageOnFocus: true,
  });

  myDiagram.groupTemplate = $(go.Group, "Vertical", {
    locationSpot: go.Spot.Bottom,
    locationObjectName: "HEADER",
    minLocation: new go.Point(0, 0),
    maxLocation: new go.Point(9999, 0),
    selectionObjectName: "HEADER",
  },
    new go.Binding("location", "loc", go.Point.parse).makeTwoWay(go.Point.stringify),
    $(go.Panel, "Auto", { name: "HEADER" },
      $(go.Shape, "Rectangle", {
        fill: $(go.Brush, "Linear", { 0: "#bbdefb", 1: go.Brush.darkenBy("#bbdefb", 0.1) }),
        stroke: null
      }),
      $(go.TextBlock, {
        margin: 5,
        font: "400 10pt Source Sans Pro, sans-serif"
      }, new go.Binding("text", "text"))
    ),
    $(go.Shape, {
      figure: "LineV",
      fill: null,
      stroke: "gray",
      strokeDashArray: [3, 3],
      width: 1,
      alignment: go.Spot.Center,
      portId: "",
      fromLinkable: false,
      fromLinkableDuplicates: false,
      toLinkable: false,
      toLinkableDuplicates: false,
      cursor: "pointer"
    },
      new go.Binding("height", "duration", computeLifelineHeight))
  );

  myDiagram.nodeTemplate = $(go.Node, {
    locationSpot: go.Spot.Top,
    locationObjectName: "SHAPE",
    minLocation: new go.Point(NaN, LinePrefix - ActivityStart),
    maxLocation: new go.Point(NaN, 19999),
    selectionObjectName: "SHAPE",
    resizable: true,
    resizeObjectName: "SHAPE",
    resizeAdornmentTemplate:
      $(go.Adornment, "Spot",
        $(go.Placeholder),
        $(go.Shape, {
          alignment: go.Spot.Bottom, cursor: "col-resize",
          desiredSize: new go.Size(6, 6), fill: "yellow"
        })
      )
  },
    new go.Binding("location", "", computeActivityLocation).makeTwoWay(backComputeActivityLocation),
    $(go.Shape, "Rectangle", {
      name: "SHAPE",
      fill: "white", stroke: "black",
      width: ActivityWidth,
      minSize: new go.Size(ActivityWidth, computeActivityHeight(0.25))
    },
      new go.Binding("height", "duration", computeActivityHeight).makeTwoWay(backComputeActivityHeight))
  );

  myDiagram.linkTemplate = $(MessageLink, {
    selectionAdorned: true, curviness: 0
  },
    $(go.Shape, "Rectangle", { stroke: "black" },
      new go.Binding("strokeDashArray", "isReturn", function (isReturn) {
        return isReturn ? [3, 3] : null;
      })),

    $(go.Shape, { stroke: "black" },
      new go.Binding("toArrow", "synchronous", function (synchronous) {
        return synchronous == 'yes' ? 'Triangle' : 'OpenTriangle';
      })),
    $(go.TextBlock, {
      font: "400 9pt Source Sans Pro, sans-serif",
      segmentIndex: 0,
      segmentOffset: new go.Point(NaN, NaN),
      isMultiline: false,
      editable: true
    },
      new go.Binding("text", "text").makeTwoWay(),
      new go.Binding("type", "text").makeTwoWay()
    )
  );

  myDiagram.addDiagramListener("LinkDrawn", function (e) {
    const link = e.subject;
    const fromNode = link.fromNode;
    const toNode = link.toNode;
    if (fromNode && toNode) {
      const fromX = fromNode.location.x;
      const toX = toNode.location.x;
      const isReturn = fromX > toX; // Comprueba la dirección de la conexión
      myDiagram.model.setDataProperty(link.data, "isReturn", isReturn);
    }
  });

  myDiagram.groupTemplateMap.add('object', $(go.Group, 'Vertical',
      {
        locationSpot: go.Spot.Bottom,
        locationObjectName: 'HEADER',
        minLocation: new go.Point(-Infinity, 129.935),
        maxLocation: new go.Point(Infinity, 0),
        selectionObjectName: 'HEADER',
        doubleClick: function (e, node) {
          // Habilita la edición de texto al hacer doble clic en el encabezado
          var textBlock = node.findObject('TEXTBLOCK');
          if (textBlock) {
            myDiagram.commandHandler.editTextBlock(textBlock);
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
            width: 130.1,
            height: 65,
          }),
        $(go.TextBlock,
          {
            margin: 0,
            font: '400 10pt Source Sans Pro, sans-serif',
            stroke: 'black',
            editable: true,
            textEdited: function (textBlock, oldText, newText) {
              var groupData = textBlock.part.data;
              myDiagram.model.setDataProperty(groupData, 'text', newText);
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

  myDiagram.groupTemplateMap.add('actor',
    $(go.Group, 'Vertical',
      {
        locationSpot: go.Spot.Bottom,
        locationObjectName: 'HEADER',
        minLocation: new go.Point(-Infinity, 129.935),
        maxLocation: new go.Point(Infinity, 0),
        selectionObjectName: 'HEADER',
        doubleClick: function (e, node) {
          // Habilita la edición de texto al hacer doble clic en el encabezado
          var textBlock = node.findObject('TEXTBLOCK');
          if (textBlock) {
            myDiagram.commandHandler.editTextBlock(textBlock);
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
            myDiagram.model.setDataProperty(groupData, 'text', newText);
          }
        },
        new go.Binding('text', 'text')),
      $(go.Shape, lineV,
        new go.Binding('height', 'duration', computeLifelineHeight))
    )
  )

  myDiagram.groupTemplateMap.add('entity',
    $(go.Group, 'Vertical',
      {
        locationSpot: go.Spot.Bottom,
        locationObjectName: 'HEADER',
        minLocation: new go.Point(-Infinity, 129.935),
        maxLocation: new go.Point(Infinity, 0),
        selectionObjectName: 'HEADER',
        doubleClick: function (e, node) {
          // Habilita la edición de texto al hacer doble clic en el encabezado
          var textBlock = node.findObject('TEXTBLOCK');
          if (textBlock) {
            myDiagram.commandHandler.editTextBlock(textBlock);
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
            myDiagram.model.setDataProperty(groupData, 'text', newText);
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

  myDiagram.groupTemplateMap.add('boundary',
    $(go.Group, 'Vertical',
      {
        locationSpot: go.Spot.Bottom,
        locationObjectName: 'HEADER',
        minLocation: new go.Point(-Infinity, 129.935),
        maxLocation: new go.Point(Infinity, 0),
        selectionObjectName: 'HEADER',
        doubleClick: function (e, node) {
          // Habilita la edición de texto al hacer doble clic en el encabezado
          var textBlock = node.findObject('TEXTBLOCK');
          if (textBlock) {
            myDiagram.commandHandler.editTextBlock(textBlock);
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
      ),
      $(go.TextBlock,
        {
          font: '400 10pt Source Sans Pro, sans-serif',
          stroke: 'black',
          editable: true,
          textEdited: function (textBlock, oldText, newText) {
            var groupData = textBlock.part.data;
            myDiagram.model.setDataProperty(groupData, 'text', newText);
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

  myDiagram.groupTemplateMap.add('control',
    $(go.Group, 'Vertical',
      {
        locationSpot: go.Spot.Bottom,
        locationObjectName: 'HEADER',
        minLocation: new go.Point(-Infinity, 129.935),
        maxLocation: new go.Point(Infinity, 0),
        selectionObjectName: 'HEADER',
        doubleClick: function (e, node) {
          // Habilita la edición de texto al hacer doble clic en el encabezado
          var textBlock = node.findObject('TEXTBLOCK');
          if (textBlock) {
            myDiagram.commandHandler.editTextBlock(textBlock);
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
            myDiagram.model.setDataProperty(groupData, 'text', newText);
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

  myDiagram.nodeTemplate = $(go.Node,
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
    new go.Binding('location', '', computeActivityLocation).makeTwoWay(backComputeActivityLocation),
    $(go.Shape, 'Rectangle',
      {
        name: 'SHAPE',
        fill: 'white',
        stroke: 'black',
        width: ActivityWidth,
        // allow Activities to be resized down to 1/4 of a time unit
        minSize: new go.Size(ActivityWidth, computeActivityHeight(0.25))
      },
      new go.Binding('height', 'duration', computeActivityHeight).makeTwoWay(backComputeActivityHeight))
  )

  myDiagram.groupTemplateMap.add("loop", $(go.Group, "Auto",
    {
      resizable: true,  // Permite redimensionar el grupo
      resizeObjectName: "SHAPE",  // El objeto que se redimensiona es el rectángulo principal
      minLocation: new go.Point(-Infinity, 140),
    },
    new go.Binding('location', 'loc', go.Point.parse).makeTwoWay(go.Point.stringify),
    $(go.Shape, "Rectangle",  // Rectángulo principal
      { name: "SHAPE", fill: "transparent", width: 250, height: 150 },
      new go.Binding("width", "width").makeTwoWay(),
      new go.Binding("height", "height").makeTwoWay()),
    $(go.TextBlock,
      {
        alignment: go.Spot.TopLeft,
        font: "bold 11pt sans-serif",
        margin: new go.Margin(5, 0, 5, 10),
        editable: false,
        text: 'loop'
      },
    ),
    $(go.TextBlock,
      {
        alignment: go.Spot.TopCenter,
        font: "11pt sans-serif",
        margin: new go.Margin(5, 0, 5, -50),
        editable: true,
      },
      new go.Binding("text", "text").makeTwoWay()),
  ))

  myDiagram.groupTemplateMap.add("alt", $(go.Group, "Vertical",
    {
      resizable: true,
      resizeObjectName: "SHAPE",
      minLocation: new go.Point(-Infinity, 140),
    },
    new go.Binding('location', 'loc', go.Point.parse).makeTwoWay(go.Point.stringify),
    $(
      go.Panel,
      'Auto',

      $(go.Shape, "Rectangle",  // Rectángulo principal
        { name: "SHAPE", fill: "transparent", width: 250, height: 150 },
        new go.Binding("width", "width").makeTwoWay(),
        new go.Binding("height", "height").makeTwoWay(),

      ),
      $(go.TextBlock,  // la etiqueta del elemento opt
        {
          alignment: go.Spot.TopLeft,
          font: "bold 11pt sans-serif",
          margin: new go.Margin(5, 0, 5, 10),
          editable: false,
          text: 'alt'
        },
      ),
      $(
        go.Panel, "Auto", // Panel para la línea punteada
        {
          height: 2,
          width: 250,
        },
        $(go.Shape, "LineH", {
          stroke: "black",
          strokeWidth: 1,
          strokeDashArray: [4, 2],
        },),

        new go.Binding("width", "width")
      ),
      $(go.TextBlock,  // la etiqueta del elemento opt
        {
          alignment: go.Spot.TopLeft,
          font: "11pt sans-serif",
          margin: new go.Margin(20, 0, 5, 10),
          editable: true,
        },
        new go.Binding("text", "text").makeTwoWay()
      ),
      $(go.TextBlock,
        {
          alignment: go.Spot.LeftCenter,
          font: "11pt sans-serif",
          margin: new go.Margin(25, 0, 0, 10),
          editable: true,
        },
        new go.Binding("text", "text2").makeTwoWay()
      ),
    ),
  ))

  myDiagram.groupTemplateMap.add("opt", $(go.Group, "Auto",
    {
      resizable: true,  // Permite redimensionar el grupo
      resizeObjectName: "SHAPE",  // El objeto que se redimensiona es el rectángulo principal
      minLocation: new go.Point(-Infinity, 140),
    },
    new go.Binding('location', 'loc', go.Point.parse).makeTwoWay(go.Point.stringify),
    $(go.Shape, "Rectangle",  // Rectángulo principal
      { name: "SHAPE", fill: "transparent" },
      new go.Binding("width", "width").makeTwoWay(),
      new go.Binding("height", "height").makeTwoWay()),
    $(go.TextBlock,  // la etiqueta del elemento opt
      {
        alignment: go.Spot.TopLeft,
        font: "bold 11pt sans-serif",
        margin: new go.Margin(5, 0, 5, 10),
        editable: false,
        text: 'opt',
      }),
    $(go.TextBlock,
      {
        alignment: go.Spot.TopCenter,
        font: "11pt sans-serif",
        margin: new go.Margin(5, 0, 5, -60),
        editable: true,
      },
      new go.Binding("text", "text").makeTwoWay()
    ),
  ))

  myDiagram.model = $(go.GraphLinksModel,
    {
      linkFromPortIdProperty: "fromPort",
      linkToPortIdProperty: "toPort",
      linkKeyProperty: "key" // Propiedad clave del enlace
    });

  await getDiagramContent();

  if (modelData) {
    myDiagram.model = go.Model.fromJson(modelData)
    if (myDiagram.model.nodeDataArray.toString()) {
      myDiagram.contentAlignment = go.Spot.Center;
    }
  }

  myDiagram.addModelChangedListener(onModelChanged);

  myDiagram.addDiagramListener("ExternalObjectsDropped", function (e) {
    var diagram = e.diagram;
    var droppedParts = e.subject;

    droppedParts.each(function (part) {
      if (part.category === 'alt' || part.category === 'opt' || part.category === 'loop') {
        var relatedElements = part.relatedElements ?? [];

        var horizontalPadding = 30;
        var bounds = part.actualBounds.copy();
        var centerX = part.actualBounds.centerX;

        bounds.width -= horizontalPadding * 2;
        bounds.setPoint(new go.Point(centerX - bounds.width / 2, bounds.y));

        var collidingElements = diagram.findObjectsIn(bounds, function (obj) {
          if (("from" in obj.part.data)) {
            return obj.part.data.category !== "alt"
              && obj.part.data.category !== "opt"
              && obj.part.data.category !== "loop"
              ? obj.part.data : null;
          }
        });

        var relatedElements = [];
        collidingElements.each(function (collidingElement) {

          if (collidingElement) {
            relatedElements.push({ 'key': collidingElement.key, 'time': collidingElement.time });
          }
        });

        if (relatedElements) {
          diagram.model.setDataProperty(part.data, "relatedElements", relatedElements);
        }

      }
    });
  });

  myDiagram.addDiagramListener("SelectionMoved", function (e) {
    var diagram = e.diagram;
    var movedParts = e.subject;

    movedParts.each(function (part) {
      if (part.category === 'alt' || part.category === 'opt' || part.category === 'loop') {
        var horizontalPadding = 30;
        var bounds = part.actualBounds.copy();
        var centerX = part.actualBounds.centerX;

        bounds.width -= horizontalPadding * 2;
        bounds.setPoint(new go.Point(centerX - bounds.width / 2, bounds.y));

        var collidingElements = diagram.findObjectsIn(bounds, function (obj) {
          if (("from" in obj.part.data)) {
            return obj.part.data.type != "fragment"
              ? obj.part.data : null;
          }
        });

        var relatedElements = [];
        collidingElements.each(function (collidingElement) {

          if (collidingElement) {
            relatedElements.push({ 'key': collidingElement.key, 'time': collidingElement.time });
          }
        });

        if (relatedElements != part.relatedElements) {
          diagram.model.setDataProperty(part.data, "relatedElements", relatedElements);
        }
      }
    });
  });

  myDiagram.grid = $(go.Panel, "Grid",
    $(go.Shape, "LineH", { strokeWidth: 0.5, strokeDashArray: [0, 9.5, 0.5, 0] })
  );

  function onModelChanged(e) {
    if (e.isTransactionFinished) {
      saveDiagramContent(true);
    }
  }
}

const initPalette = () => {
  const myPalette = $(
    go.Palette,
    'myPaletteDiv',
    {
      scrollsPageOnFocus: false,
      nodeTemplateMap: myDiagram.nodeTemplateMap,
      groupTemplateMap: myDiagram.groupTemplateMap,
      initialScale: 0.75,
      //initialPosition: new go.Point(500, 400),
      padding: new go.Margin(100, 0, 0, 0),
      layout: $(go.GridLayout,
        { wrappingColumn: 3, alignment: go.GridLayout.Position, cellSize: new go.Size(50, 50) })
    }
  )

  myPalette.model = new go.GraphLinksModel([
    { category: 'actor', key: 'actor', text: 'Actor', isGroup: true, duration: 10 },
    { category: 'object', key: 'object', text: 'Object', isGroup: true, duration: 10 },
    { category: 'entity', key: 'entity', text: 'Entity', isGroup: true, duration: 10 },
    { category: 'boundary', key: 'boundary', text: 'Boundary', isGroup: true, duration: 10 },
    { category: 'control', key: 'control', text: 'Control', isGroup: true, duration: 10 },
    { category: 'alt', key: 'alt', text: '[condition]', text2: '[condition]', isGroup: true, relatedElements: [], width: 250, height: 150 },
    { category: 'opt', key: 'opt', text: '[condition]', isGroup: true, relatedElements: [], width: 250, height: 150 },
    { category: 'loop', key: 'loop', text: '[condition]', isGroup: true, relatedElements: [], width: 250, height: 150 },
  ])
}
</script>

<style>
@import '@/Assets/styles/secuence-diagram.css';
</style>
