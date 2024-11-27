<template>
  <div>
    <canvas ref="webglRef" class="webgl-renderer"></canvas>
    <div ref="cssRef" class="css-renderer"></div>
  </div>
</template>

<script>
import { Group, Object3D, PlaneGeometry } from "three";
import { MeshBasicMaterial } from "three";
import { Matrix4 } from "three";
import { Vector2 } from "three";
import { Raycaster } from "three";
import { Frustum } from "three";
import { TextureLoader } from "three";
import { Mesh } from "three";
import { Scene, OrthographicCamera, WebGLRenderer } from "three"
import { CSS3DObject, CSS3DRenderer, OrbitControls } from "three/examples/jsm/Addons.js";
import { labels } from "./data/labels";

class CustomControl extends OrbitControls {
  constructor(camera, domElem){
    super(camera, domElem)
    this.mouseButtons.LEFT = 2;
    this.mouseButtons.RIGHT = 0; // doesn't do anything???
    this.touches.ONE = 1; // Pan, I think
    this.touches.TWO = 3; // Zoom
  }
}

function buildHtmlElement(document, label){
  // const obj = new Object3D()
  const element = document.createElement(label?.elemType ?? "div")
  element.id = label.id
  element.textContent = label.text
  element.style.padding = "10px"
  element.style.backgroundColor = "white"
  element.style.color = "black"
  element.style.transition = "all 0.1s ease"
  element.addEventListener("mouseover", () => {
    element.style.backgroundColor = "black"
    element.style.color = "white"
  })
  element.addEventListener("mouseout", () => {
    element.style.backgroundColor = "white"
    element.style.color = "black"
  })
  if (label.onClick) {
    element.addEventListener("mousedown", label.onClick)
  }
  const cssObject = new CSS3DObject(element)
  cssObject.position.x = label.x
  cssObject.position.y = label.y
  cssObject.position.z = 0.012
  cssObject.scale.x = 0.05
  cssObject.scale.y = 0.05
  cssObject.visible = true
  // obj.add(cssObject)
  return cssObject
}

export default {
  setup(){
    const webglRef = ref()
    const cssRef = ref()
    const thewholething = {
      scene: null,
      camera: null,
      renderer: null,
      labelRenderer: null,
      controls: null,
      zoom: {
        min: 0,
        max: 3,
        currentZoom: 0
      },
      canvas: null
    }

    onMounted(() => {
      const maxWidth = Math.max(document.documentElement.clientWidth, document.body.clientWidth, window.innerWidth || 0);
      const maxHeight = Math.max(document.documentElement.clientHeight, document.body.clientHeight, window.innerHeight || 0);
      const aspect = maxWidth / maxHeight;
      const maxSize = 64

      thewholething.scene = new Scene();
      thewholething.camera = new OrthographicCamera(
        maxSize / -2, 
        maxSize / 2,
        maxSize / aspect / 2,
        maxSize / aspect / -2,
        thewholething.zoom.min,
        thewholething.zoom.max
      );
      thewholething.camera.position.z = thewholething.zoom.min + 1;
      thewholething.camera.position.x = 0
      thewholething.camera.position.y = 0
      thewholething.camera.zoom = 1

      thewholething.canvas = webglRef.value
      thewholething.renderer = new WebGLRenderer({
        canvas: thewholething.canvas,
        alpha: false,
        antialias: true
      })
      thewholething.renderer.setPixelRatio(window.devicePixelRatio)
      thewholething.renderer.setSize(window.innerWidth, window.innerHeight);

      thewholething.labelRenderer = new CSS3DRenderer()
      thewholething.labelRenderer.domElement.style.position = "absolute";
      thewholething.labelRenderer.domElement.style.top = "0px";
      thewholething.labelRenderer.domElement.style.left = '0px';
      thewholething.labelRenderer.setSize(window.innerWidth, window.innerHeight);
      cssRef.value.appendChild(thewholething.labelRenderer.domElement);

      thewholething.controls = new CustomControl(thewholething.camera, thewholething.labelRenderer.domElement)
      thewholething.controls.enableDamping = true;
      thewholething.controls.dampingFactor = 0.1;
      thewholething.controls.screenSpacePanning = true;
      thewholething.controls.minZoom = thewholething.zoom.min + 1;
      thewholething.controls.maxZoom = thewholething.zoom.max;
      thewholething.controls.panSpeed = 0.8;
      thewholething.controls.zoomSpeed = 1;
      thewholething.controls.enableRotate = false;

      const material = new MeshBasicMaterial({ transparent: true, opacity: 0 })
      const tiles = []

      const buildMeshes = function(tileSize, zLevel){
        const meshes = []
        const planeGeometry = new PlaneGeometry(tileSize, tileSize, 1, 1)
        const maxIndex = maxSize / tileSize;
        for (let xCoord = 0; xCoord < maxIndex; xCoord++) {
          for (let yCoord = 0; yCoord < maxIndex; yCoord++) {
            const mesh = new Mesh(planeGeometry, material)
            // 64 / 2 - 32 / 2 - 32 * 0 => 32 - 16
            mesh.position.y = maxSize / 2 - tileSize / 2 - tileSize * yCoord
            mesh.position.x = xCoord * tileSize - maxSize / 2 + tileSize / 2
            mesh.position.z = 0.012
            meshes.push({
              loaded: false,
              mesh,
              textureUrl: `http://localhost:3000/map/${zLevel + 1}-${xCoord + 1}-${yCoord + 1}.jpg`,
              id: `${zLevel + 1}-${xCoord + 1}-${yCoord + 1}`,
              zoomGroup: zLevel,
              posX: yCoord * tileSize - maxSize / 2 + tileSize / 2,
              posY: maxSize / 2 - tileSize / 2 - tileSize * xCoord
            })
          }
        }

        return meshes
      }

      for (let z = 0; z <= thewholething.zoom.max; z++) {
        const myTileSize = maxSize / (2 ** z)
        /*
          actual size: 64
          z 0: tc 1, ts 64
          z 1: tc 4, ts 32
          z 2: tc 16, ts 16
          z 3: tc 64, ts 8
        */
        tiles.push(...buildMeshes(myTileSize, z))
      }

      const textureLoader = new TextureLoader();

      const frustum = new Frustum()

      const updateScene = () => {
        frustum.setFromProjectionMatrix(new Matrix4().multiplyMatrices(thewholething.camera.projectionMatrix, thewholething.camera.matrixWorldInverse))
        tiles.map(tile => {
          const isIntersecting = frustum.intersectsObject(tile.mesh)
          if (tile.zoomGroup <= thewholething.camera.zoom && !tile.loaded && isIntersecting) {
            textureLoader.load(tile.textureUrl, function(texture){
              tile.mesh.material = new MeshBasicMaterial({
                map: texture
              })
              tile.mesh.material.opacity = 1;
              tile.mesh.material.needsUpdate = true;
              tile.mesh.visible = true;
              tile.loaded = true;
            })
          }
        })
      }
      
      updateScene();
      
      const raycaster = new Raycaster()
      const touchVector = new Vector2(0, 0)

      cssRef.value.addEventListener("mousemove", (e) => {
        touchVector.x = e.clientX / thewholething.renderer.domElement.clientWidth * 2 - 1;
        touchVector.y = -e.clientY / thewholething.renderer.domElement.clientHeight * 2 + 1;
      })
      cssRef.value.addEventListener("wheel", updateScene)
      cssRef.value.addEventListener("click", () => {
        updateScene()
        raycaster.setFromCamera(touchVector, thewholething.camera)
        tiles.map(tile => {
          const intersection = raycaster.intersectObject(tile.mesh, false)
          if (intersection.length > 0) {
            // check for intersection here, I guess
            console.log({ intersection })
          }
        })
      })
      tiles.forEach(tile => thewholething.scene.add(tile.mesh))
      const threeDLabels = labels.map(label => {
        return buildHtmlElement(document, label)
      })
      const labelsGroup = new Group()
      threeDLabels.forEach(l => labelsGroup.add(l))
      thewholething.scene.add(labelsGroup)

      const animate = function(){
        thewholething.controls.update();
        thewholething.renderer.render(thewholething.scene, thewholething.camera);
        thewholething.labelRenderer.render(thewholething.scene, thewholething.camera);
        requestAnimationFrame(animate);
      }

      animate()
    })

    return {
      webglRef,
      cssRef
    }
  }
}
</script>

<style scoped>
.webgl-renderer {
  position: absolute;
  width: 100%;
  height: 100%;
  top: 0;
  left: 0;
}
.css-renderer {
  position: absolute;
  width: 100%;
  height: 100%;
  top: 0;
  left: 0;
}
</style>

<style>
body {
  margin: 0;
  padding: 0;
}
</style>