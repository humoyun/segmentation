<template>
  <div class="segmentation-page">
    <span>Segmentation</span>

    <div class="workspace">
      <div class="canvas-panel">
        <canvas id="segmentation-canvas" width="1200" height="800" />
      </div>

      <div class="toolbar">
        <div class="controls">
          <h3>Controls</h3>
          <div class="modes">
            <el-radio-group
              v-model="control"
              @change="handleControlChange"
              :disabled="!currentImage"
            >
              <el-radio-button label="select"></el-radio-button>
              <el-radio-button label="brush"></el-radio-button>
              <el-radio-button label="erase"></el-radio-button>
              <el-radio-button label="pan"></el-radio-button>
            </el-radio-group>
          </div>

          <div>
            <el-button
              type="primary"
              plain
              @click="handleZoomToFit()"
              :disabled="!currentImage"
            >
              Zoom to fit
            </el-button>
            <el-button type="success" @click="save()">Save</el-button>
            <el-button type="info" @click="download()">Download</el-button>
          </div>

          <div class="uploaders">
            <el-upload
              class="image-uploader"
              action="#"
              :on-change="handleImage"
            >
              <el-button type="warning">Upload image</el-button>
            </el-upload>

            <el-upload class="mask-loader" action="#" :on-change="loadBitmask">
              <el-button type="warning">Load bitmask</el-button>
            </el-upload>
          </div>

          <div style="width: 200px">
            <el-slider
              v-model="brushRadius"
              @change="handleBrushRadiusChange"
              :min="5"
              :max="50"
            />
          </div>

          <div class="modes">
            <el-radio-group
              v-model="mode"
              @change="handleModeChange"
              :disabled="!currentImage"
            >
              <el-radio-button label="image"></el-radio-button>
              <el-radio-button label="both"></el-radio-button>
              <el-radio-button label="label"></el-radio-button>
            </el-radio-group>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { fabric } from "fabric";

// fabric.Object.prototype.transparentCorners = false;

/**
To save a 4-channel image as a PNG file using JavaScript, you can use the HTMLCanvasElement.toDataURL() 
method to get a base64-encoded data URI representing the image, and then use the download attribute of 
an anchor element to prompt the user to download the image.

Here is an example of how you might use these techniques to save a 4-channel image as a PNG file:

<!-- HTML -->
<canvas id="canvas"></canvas>
<a id="download-link" download>Download Image</a>

<!-- JavaScript -->
const canvas = document.getElementById('canvas');
const downloadLink = document.getElementById('download-link');

// Create a 4-channel image on the canvas
const context = canvas.getContext('2d');
const imageData = context.createImageData(canvas.width, canvas.height);

for (let i = 0; i < imageData.data.length; i += 4) {
  // Set the values for the 4 channels of the pixel
  imageData.data[i] = 255; // Red channel
  imageData.data[i + 1] = 0; // Green channel
  imageData.data[i + 2] = 0; // Blue channel
  imageData.data[i + 3] = 255; // Alpha channel
}

context.putImageData(imageData, 0, 0);

// Get the data URI of the image
const dataURI = canvas.toDataURL('image/png');

// Set the href of the download link to the data URI
downloadLink.href = dataURI;

// Set the download attribute of the link to the desired file name
downloadLink.download = 'image.png';

// Simulate a click on the download link
downloadLink.click();
 */

export const classColors = [
  "#ffb319",
  "#e3108f",
  "#f5ed05",
  "#491ddb",
  "#05f5b9",
  "#f5051d",
  "#21f505",
  "#0f4f40",
  "#bd59eb",
  "#804f05",
];

// https://segmentsai-prod.s3.eu-west-2.amazonaws.com/assets/admin-tobias/f31fbb68-e644-4311-bb03-f4d2d7285a33.jpg
export default {
  name: "SegmentationPanel",

  data() {
    return {
      mode: "both",
      control: "select",
      brushRadius: 25,
      canvas: null,
      pressedKeys: {},
      currentImage: null,
      colorCounter: 0,
    };
  },

  methods: {
    init() {
      console.log("INIT");

      this.canvas = new fabric.Canvas("segmentation-canvas");
      const c = this.canvas;
      this.canvas.on("mouse:wheel", (opt) => {
        const controlKey = this.pressedKeys["Control"];

        if (controlKey === "keydown") {
          const delta = opt.e.deltaY;
          let zoom = this.canvas.getZoom();
          zoom *= 0.999 ** delta;
          if (zoom > 20) zoom = 20;
          if (zoom < 0.01) zoom = 0.01;

          const center = c.getCenter();

          const centerPoint = new fabric.Point(center.left, center.top);
          c.zoomToPoint(centerPoint, zoom);
          // need to fix case `zoom to fit`, not working as expected
          // c.zoomToPoint({ x: opt.e.offsetX, y: opt.e.offsetY }, zoom);
          opt.e.preventDefault();
          opt.e.stopPropagation();
        }
      });

      this.canvas.on("object:selected", (obj) => {
        console.log("object:selected ", obj);
      });

      this.canvas.on("path:created", (path) => {
        console.log("path:created ", path);
      });

      // this.canvas.freeDrawingBrush = new fabric.PencilBrush(this.canvas);

      this.canvas.on("mouse:up", (e) => {
        console.log("eL: ", e);
        console.log("paths", this.canvas.getObjects());
      });

      this.canvas.on("mouse:over", (e) => {
        if (
          this.control === "select" &&
          e.target &&
          e.target.type === "group"
        ) {
          e.target.getObjects().forEach((pathObj) => {
            pathObj.set({
              stroke: e.target.classColor,
              opacity: 0.5,
            });
          });
          this.canvas.renderAll();
        }
      });

      this.canvas.on("mouse:out", (e) => {
        if (
          this.control === "select" &&
          e.target &&
          e.target.type === "group"
        ) {
          e.target.getObjects().forEach((pathObj) => {
            pathObj.set({
              stroke: e.target.classColor,
              opacity: 0.7,
            });
          });
          this.canvas.renderAll();
        }
      });

      this.canvas.on("mouse:move", () => {
        this.canvas.getObjects().forEach((obj) => {
          // o.forEachObject(function (obj) {

          if (obj.isFillable) {
            // obj.set({
            //   stroke: "TODO",
            //   fill: "TODO",
            // });
          } else {
            // obj.set({
            //   stroke: "TODO",
            //   fill: "transparent",
            // });
          }

          // });
        });
        this.canvas.renderAll();
      });
    },

    handleZoomToFit() {
      const center = this.canvas.getCenter();
      const centerPoint = new fabric.Point(center.left, center.top);
      this.canvas.zoomToPoint(centerPoint, 1);
      this.canvas.renderAll();
    },

    handleModeChange(mode) {
      if (mode === "label") {
        const image = new fabric.Image("");
        this.canvas.setBackgroundImage(
          image,
          this.canvas.renderAll.bind(this.canvas)
        );
        this.canvas.backgroundColor = "#000";

        this.canvas.getObjects().forEach((obj) => {
          obj.set({
            visible: true,
          });
        });
      }
      if (mode === "image") {
        this.drawBgImage(this.currentImage);
        this.canvas.backgroundColor = "#fff";

        this.canvas.getObjects().forEach((obj) => {
          obj.set({
            visible: false,
          });
        });
      }

      if (mode === "both") {
        this.drawBgImage(this.currentImage);
        this.canvas.backgroundColor = "#fff";
        this.canvas.getObjects().forEach((obj) => {
          obj.set({
            visible: true,
          });
        });
      }

      this.canvas.renderAll();
    },

    combinePaths(paths) {
      if (!paths.length) {
        return null;
      }

      let singlePath = paths[0];
      for (let i = 1; i < paths.length; i++) {
        singlePath = [...singlePath, ...paths[i]];
      }
      return singlePath;
    },

    handleControlChange(control) {
      switch (control) {
        case "select":
          {
            console.log("select");
            this.canvas.select = true;
            this.canvas.isDrawingMode = false;
          }
          break;

        case "brush":
          {
            console.log("brush");
            this.canvas.isDrawingMode = true;
            this.canvas.select = false;
            this.canvas.freeDrawingBrush = new fabric.PencilBrush(this.canvas);
            this.canvas.freeDrawingBrush.width = 25;
            this.canvas.freeDrawingBrush.color = "rgba(255, 255, 255, 0.6)";
          }
          break;

        case "erase":
          {
            console.log("erase");
            this.canvas.freeDrawingBrush = new fabric.EraserBrush(this.canvas);
            this.canvas.isDrawingMode = true;
            this.canvas.select = false;
          }
          break;

        default:
          break;
      }

      if (control !== "brush") {
        // iterate over all paths and merge them into one unified group
        const objects = this.canvas.getObjects();

        if (objects) {
          const paths = objects.filter((o) => o.type === "path");
          console.log("paths; ", paths);
          console.log(
            "objects; ",
            objects.filter((o) => o.type === "group")
          );
          const group = new fabric.Group([...paths], {
            hoverCursor: "pointer",
            selectable: false,
            subTargetCheck: true,
          });
          // remove all existing path objects first
          group.getObjects().forEach((path) => {
            this.canvas.remove(path);
          });

          group.getObjects().forEach((obj) => {
            obj.set({
              cornerStrokeColor: "green",
              stroke: classColors[this.colorCounter],
              opacity: 0.7,
            });
          });
          group.classColor = classColors[this.colorCounter];
          // then add unified group
          this.canvas.add(group);
          this.colorCounter++;
          console.log("colorCounter ", this.colorCounter);
          this.canvas.renderAll();
        }
      }

      // this.canvas.isDrawingMode = true;
      // this.canvas.freeDrawingBrush = new fabric.PencilBrush(this.canvas);
      // this.canvas.freeDrawingBrush.width = 20;
    },

    handleImage(file) {
      this.drawBgImage(file.raw);
      this.currentImage = file.raw;
    },

    drawBgImage(bgImage) {
      const reader = new FileReader();
      const c = this.canvas;

      reader.onload = (e) => {
        const imgObj = new Image();
        imgObj.src = e.target.result;

        imgObj.onload = () => {
          let ratio = 1;
          if (imgObj.width < imgObj.height) {
            ratio = c.height / imgObj.height;
          } else if (imgObj.width > imgObj.height) {
            ratio = c.width / imgObj.width;
          } else if (imgObj.width > c.width) {
            ratio = c.width / imgObj.width;
          } else if (imgObj.height > c.height) {
            ratio = c.height / imgObj.height;
          }

          const imgInstance = new fabric.Image(imgObj, {
            scaleX: ratio,
            scaleY: ratio,
          });
          const center = c.getCenter();
          c.setBackgroundImage(imgInstance, c.renderAll.bind(this.canvas), {
            top: center.top,
            left: center.left,
            originX: "center",
            originY: "center",
          });

          console.log("imgInstance", imgInstance);
          this.canvas.renderAll();
        };
      };

      reader.readAsDataURL(bgImage);
      // console.log(URL.createObjectURL(bgImage));
    },

    handleBrushRadiusChange(value) {
      this.canvas.freeDrawingBrush.width = value;
    },

    download() {
      const image = this.canvas.toDataURL({
        format: "png",
      });

      const link = document.createElement("a");
      link.setAttribute("download", "mask.png");
      link.href = image;
      link.click();
    },

    loadBitmask(file) {
      console.log("load ", file.raw);
    },

    save() {},
  },

  mounted() {
    this.init();
    document.addEventListener("keydown", (e) => {
      this.pressedKeys[e.key] = e.type;
    });
    document.addEventListener("keyup", (e) => {
      this.pressedKeys[e.key] = e.type;
    });
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}

.workspace {
  display: flex;
  border: 1px solid #42b983;
}

#segmentation-canvas {
}

.toolbar {
  width: 100%;
  background-color: #eef1fb;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.uploaders {
  display: flex;
  gap: 10px;
}

.controls {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 20px;
  align-items: center;
}
</style>
