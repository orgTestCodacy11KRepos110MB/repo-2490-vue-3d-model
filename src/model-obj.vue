<script lang="ts">
import { defineComponent } from 'vue';
import { Object3D } from 'three';
import { OBJLoader } from 'three/examples/jsm/loaders/OBJLoader';
import { MTLLoader } from 'three/examples/jsm/loaders/MTLLoader';
import { DDSLoader } from 'three/examples/jsm/loaders/DDSLoader';
import { LoadingManager } from 'three/src/loaders/LoadingManager';
import { toIndexed } from './utils';
import mixin from './model-mixin.vue';

// TODO: Better way to handle texture formats
const manager = (new LoadingManager()); // 0.122+ new api
manager.addHandler(/\.dds$/i, new DDSLoader());

export default defineComponent({
  name: 'model-obj',
  mixins: [mixin],
  props: {
    lights: {
      type: Array,
      default: () => {
        return [
          {
            type: 'HemisphereLight',
            position: { x: 0, y: 1, z: 0 },
            skyColor: 0xaaaaff,
            groundColor: 0x806060,
            intensity: 0.2,
          },
          {
            type: 'DirectionalLight',
            position: { x: 1, y: 1, z: 1 },
            color: 0xffffff,
            intensity: 0.8,
          },
        ];
      },
    },
    smoothing: {
      type: Boolean,
      default: false,
    },
    mtlPath: {
      type: String,
    },
    mtl: {
      type: String,
    },
  },
  data(this: any) {
    const objLoader = new OBJLoader(manager);
    const mtlLoader = new MTLLoader(manager);

    mtlLoader.setCrossOrigin(this.crossOrigin);
    mtlLoader.setRequestHeader(this.requestHeader);
    objLoader.setRequestHeader(this.requestHeader);

    return {
      loader: objLoader,
      mtlLoader,
    };
  },
  watch: {
    mtl() {
      this.load();
    },
  },
  methods: {
    process(object: Object3D) {
      if (this.smoothing) {
        object.traverse((child: any) => {
          if (child.geometry) {
            child.geometry = toIndexed(child.geometry);
            child.geometry.computeVertexNormals();
          }
        });
      }
    },
    load(this: any) {
      if (!this.src) return;

      if (this.object) {
        this.wrapper.remove(this.object);
      }

      const onLoad = (object: Object3D) => {
        this.reportProgress('end');
        if (this.process) {
          this.process(object);
        }

        this.addObject(object);

        this.$emit('load');
      };

      const onProgress = (event: ProgressEvent) => {
        this.reportProgress('progress', event);
        this.$emit('progress', event);
      };

      const onError = (event: ErrorEvent) => {
        this.reportProgress('end');
        this.$emit('error', event);
      };

      this.reportProgress('start');

      if (this.mtl) {
        let { mtlPath } = this;
        let mtlSrc = this.mtl;

        if (!this.mtlPath) {
          const result = /^(.*\/)([^/]*)$/.exec(this.mtl);

          if (result) {
            mtlPath = result[1];
            mtlSrc = result[2];
          }
        }

        if (mtlPath) {
          this.mtlLoader.setPath(mtlPath);
        }

        this.mtlLoader.load(mtlSrc, (materials: any) => {
          materials.preload();

          this.loader
            .setMaterials(materials)
            .load(this.src!, onLoad, onProgress, onError);
        }, () => {}, onError);
      } else {
        this.loader.load(this.src, onLoad, onProgress, onError);
      }
    },
  },
});
</script>
