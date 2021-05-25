<template>
  <div
    ref="wrapper"
    class="plan-editor-container"
    :style="cursor"
  >
    <svg
      id="svg-plan"
      ref="container"
      xmlns="http://www.w3.org/2000/svg"
      class="container-svg"
      :viewBox="viewBox"
      preserveAspectRatio="xMidYMid meet"
      :style="planStyle"
      @mouseup="onMouseUp($event)"
      @mousemove="handleMouseMove($event)"
      @mousedown="startDrag($event)"
    >
      <image
        width="100%"
        height="100%"
        :style="imageStyle"
        :href="path"
        @click="selected = null"
      />
      <slot />
      <template v-if="curves">
        <ServerCurves
          v-for="curve in curves"
          :key="curve.id"
          :curve="curve"
          :selected="selected"
          @select="selectPoly($event)"
          @localUpd="moveDot($event)"
        />
      </template>
      <template v-if="points.length">
        <polyline
          id="drawPolyline"
          :points="getPoints"
          :stroke="mode === 'DRAW' ? '#000000' : 'none'"
          :fill="mode !== 'DRAW' ? currColor : 'none'"
          :class="{ 'drawingMode': mode === 'DRAW' }"
        />

        <polygon
          v-show="mode === 'FINISH'"
          :points="getPoints"
          stroke="#000"
          fill="none"
        />
      </template>

      <circle
        v-for="(item, index) in points"
        :key="index"
        :cx="item[0]"
        :cy="item[1]"
        r="8"
      />
      <line
        v-if="points.length >= 1 && mode === 'DRAW'"
        :x1="points[points.length - 1][0]"
        :y1="points[points.length - 1][1]"
        :x2="mouseCoordinates[0]"
        :y2="mouseCoordinates[1]"
        stroke="#001"
        stroke-width="1"
      />
    </svg>
    <ActionMenu
      ref="actionMenu"
      :key="polyTarget"
      persistent
      :transition-show="''"
      :name="selectedName"
      :target="polyTarget"
      v-model="showActions"
      @assign="$emit('assign', $event)"
      @remove="$emit('remove', curCurve)"
    />

    <component
      :is="closePopupComponent"
      v-if="points.length"
      :items="markupList"
      persistent
      :rendered-items="renderedList"
      :value="show"
      @select="create($event)"
      @cancel="cancel()"
    />
  </div>
</template>

<script lang="ts">
import 'reflect-metadata';
import {
  Component, Prop, Watch,
} from 'vue-property-decorator';
import {
  ICurve, IMarkupList, ICircleInfo, IFloorObj, IBuilding, IRender,
} from 'src/store/modules/models';
import { Coordinates } from 'src/utils/compareCoordinates';
import EditorClosePopup from 'src/admin/components/Popups/EditorClosePopup.vue';
import { mapFields } from 'vuex-map-fields';
import ServerCurves from 'src/admin/components/ServerCurves.vue';
import ActionMenu from 'src/admin/components/Popups/ActionMenu.vue';
import { QMenu } from 'quasar';
import Complex from 'src/store/modules/Complex';
import { getModule } from 'vuex-module-decorators';
import FloorClosePopup from 'src/admin/components/Popups/FloorClosePopup.vue';
import Color from 'src/store/modules/Color';
import Canvas from 'components/base/curves/Canvas.vue';

@Component({
  components: {
    FloorClosePopup, ActionMenu, ServerCurves, EditorClosePopup,
  },
  computed: mapFields(['Editor.mode', 'Editor.opacity']),
})
export default class SvgEditor extends Canvas {
  name = 'svg-editor'
  @Prop() path: string | undefined
  @Prop() type: string | undefined
  // @Prop() curves: ICurve | undefined
  @Prop() markupList!: Array<IMarkupList>
  @Prop() AllItems!: Array<IFloorObj>
  @Prop() renderedList!: Array<number>
  public mode!: string;
  public opacity!: number;

  color = getModule(Color, this.$store)

  get currColor(): string {
    return this.color.activeColor;
  }

  get closePopupComponent() {
    switch (this.type) {
      default:
        return 'EditorClosePopup';
      case 'Floor':
        return 'FloorClosePopup';
    }
  }

  get getPoints() {
    return this.points.flat(1);
  }

  get imageStyle() {
    return { opacity: ${this.opacity / 100} };
  }

get selectedItem():IFloorObj | IBuilding | IRender | null {
    if (!this.selected) {
      return null;
    }
    switch (this.type) {
      case 'Floor':
        return this.AllItems.find((e) => e.id === this.curCurve?.spaceId) as IFloorObj;
      case 'Building':
        return this.markupList.find((e) => e.id === this.curCurve?.floor?.id) as IBuilding;

      default:
        return this.markupList.find((e) => e.id === this.curCurve?.building) as IRender;
    }
  }

  points: number[][] = [];

  svg: any | null = null
  pt: any = null
  mouseCoordinates: Array<number> = [];

  show = false
  selected: number | null = null
  curCurve: ICurve | null = null
  showActions = false
  polyTarget: boolean | string = false

  complex = getModule(Complex, this.$store)

  get cursor(): string {
    return this.scale >= this.minScaleSize && this.mode !== 'DRAW' ? 'cursor:move' : '';
  }

  async moveDot(e: any) {
    this.dragging = e.drag;
    const payload: ICircleInfo = {
      coords: this.mouseCoordinates,
      ...e,
    };
    await this.complex.updatePoints(payload);
  }

  get selectedName() {
    if (this.type === 'Floor') {
      return ${this.selectedItem?.number as number} помещение;
    }
    return this.selectedItem?.name as string;
  }

  keyboardNavigation() {
    document.addEventListener('keydown', (e: KeyboardEvent) => {
      const { key } = e;
      if (key === 'Escape' && this.mode === 'DRAW') {
        this.finish();
      }
    });
  }

  create(id: number) {
    this.$emit('create', { id, points: this.points });
    this.points.splice(0, this.points.length);
    this.show = false;
    this.mode = 'DRAW';
  }

  cancel() {
    this.points.splice(0, this.points.length);
    this.show = false;
    this.mode = 'DRAW';
  }

  @Watch('selected')
  onSelectedChanged(val:number) {
    if (!val) {
      this.polyTarget = false;
    }

    this.$nextTick(() => {
      (this.$refs.actionMenu as QMenu).updatePosition();
    });

    this.showActions = !!val;
  }

  @Watch('mode')
  onModeChanged(val: string) {
    if (val === 'SELECT') {
      this.points.splice(0, this.points.length);
    }
  }

  selectPoly({ id, curve }:{id: number, curve: ICurve}) {
    if (this.mode !== 'SELECT') {
      return;
    }

    if (id === this.selected) {
      this.selected = null;
    } else {
      this.selected = id;
      this.curCurve = curve;

      this.polyTarget = #curve-polygon-${id};
      this.$nextTick(() => {
        (this.$refs.actionMenu as QMenu).updatePosition();
      });
    }
  }

  onMouseUp() {
    this.stopDrag();
    if (this.mode !== 'DRAW') {
      return;
    }
    if (this.renderedList?.length >= this.markupList?.length) {
      this.$q.notify({ message: 'Вы уже разметили все активны элементы, переходим в режим выбора' });
      this.mode = 'SELECT';
      return;
    }

    if (this.points.length > 2 && Coordinates.compare(this.mouseCoordinates, this.points[0])) {
      this.finish();
    } else {
      this.points.push(this.mouseCoordinates);
    }
  }

  handleMouseMove(e:MouseEvent) {
    // if (e.target.nodeName !== 'image') {
    //   return;
    // }

    this.onDrag(e);

    this.pt.x = e.clientX;
    this.pt.y = e.clientY;

    const cursorpt = this.pt.matrixTransform(this.svg.getScreenCTM().inverse());
    const { x } = cursorpt;
    const { y } = cursorpt;
    this.mouseCoordinates = [x, y];
  }

  finish() {
    this.mode = 'FINISH';
    this.show = true;
  }

  @Watch('$route.fullPath')
  onRouteChange() {
    this.getImageDimensions();
    this.resetZoom();
  }

  mounted() {
    this.getImageDimensions();
    this.keyboardNavigation();
    this.$nextTick(() => {
      this.svg = document.getElementById('svg-plan');
      this.pt = this.svg?.createSVGPoint();
    });
  }

  getImageDimensions() {
    const img = new Image();

    img.onload = () => {
      this.imageRect.height = img.height;
      this.imageRect.width = img.width;
    };
    if (this.path) {
      img.src = this.path;
    }
  }
}
</script>

Sasha Staetskiy, [07.10.20 15:24]
<style lang="scss" scoped>
.canvas {
  border: 1px solid #000;
  margin: 0 auto;
  min-height: 550px;
  width: 100%;
}

.plan-editor {
  align-items: center;
  display: flex;
  height: 100%;
  justify-content: center;

  &-container {
    // align-items: center;

    border: 1px solid #a21717;
    // display: flex;

    height: 75vh;
    margin: 0 auto;
    overflow: hidden;
    position: relative;
    width: 80vw;
  }
}

.container-svg {
  display: block;
  height: 100%;
  margin: 0 auto;
  position: relative;
  width: 100%;

  image {
    opacity: 1;
  }

  ::v-deep {
    image,
    circle,
    g,
    polyline,
    polygon {
      position: relative;
    }
  }
}

.drawing-mode {
  fill: none;
}

</style>

<style>
  circle {
    cursor: grab;
  }
</style>
