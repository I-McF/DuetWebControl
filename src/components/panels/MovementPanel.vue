<style>
.move-btn {
  padding-left: 0;
  padding-right: 0;
  min-width: 0;
}

.center-menu-item > div {
  justify-content: center;
}
</style>

<template>
  <v-card>
    <v-card-title class="pt-2 pb-0">
      <v-icon small class="mr-1">swap_horiz</v-icon>
      {{ $t('panel.movement.caption') }}
    </v-card-title>

    <v-card-text class="pt-0 pb-2">
      <!-- Mobile Buttons -->
      <v-flex grow class="hidden-md-and-up">
        <v-menu full-width offset-y :disabled="uiFrozen" v-tab-control>
          <template slot="activator">
            <v-btn block color="primary" class="mx-0" :disabled="uiFrozen">
              {{ $t('panel.movement.compensation') }}
              <v-icon>arrow_drop_down</v-icon>
            </v-btn>
          </template>

          <v-card>
            <v-list>
              <template v-if="move.compensation">
                <v-list-tile
                  class="center-menu-item"
                >{{ $t('panel.movement.compensationInUse', [move.compensation]) }}</v-list-tile>

                <v-divider></v-divider>
              </template>

              <v-list-tile @click="sendCode('G32')">
                <v-icon class="mr-1">view_module</v-icon>
                {{ $t(move.geometry.type === 'delta' ? 'panel.movement.runDelta' : 'panel.movement.runBed') }}
              </v-list-tile>
              <v-list-tile
                :disabled="!move.compensation || move.compensation.indexOf('Point') === -1"
                @click="sendCode('M561')"
              >
                <v-icon class="mr-1">clear</v-icon>
                {{ $t('panel.movement.disableBedCompensation') }}
              </v-list-tile>

              <v-divider></v-divider>

              <v-list-tile @click="sendCode('G29')">
                <v-icon class="mr-1">grid_on</v-icon>
                {{ $t('panel.movement.runMesh') }}
              </v-list-tile>
              <v-list-tile @click="showMeshEditDialog = true">
                <v-icon class="mr-1">edit</v-icon>
                {{ $t('panel.movement.editMesh') }}
              </v-list-tile>
              <v-list-tile @click="sendCode('G29 S1')">
                <v-icon class="mr-1">save</v-icon>
                {{ $t('panel.movement.loadMesh') }}
              </v-list-tile>
              <v-list-tile :disabled="move.compensation !== 'Mesh'" @click="sendCode('G29 S2')">
                <v-icon class="mr-1">grid_off</v-icon>
                {{ $t('panel.movement.disableMeshCompensation') }}
              </v-list-tile>
            </v-list>
          </v-card>
        </v-menu>
      </v-flex>
      <v-layout justify-center row wrap class="hidden-md-and-up">
        <v-layout column>
          <v-flex>
            <code-btn
              color="primary"
              code="G28"
              :title="$t('button.home.titleAll')"
              block
            >{{ $t('button.home.captionAll') }}</code-btn>
          </v-flex>
          <v-flex>
            <code-btn
              color="primary"
              code="`G10  L20 X0 Y0 Z0\nM500`"
              :title="$t('button.work.titleAll')"
              block
            >{{ $t('button.work.captionAll') }}</code-btn>
          </v-flex>
        </v-layout>
        <v-layout column>
          <v-layout justify-center row wrap class="hidden-md-and-up">
            <v-flex shrink class="px-1"></v-flex>
            <v-flex v-for="axis in displayedAxes" :key="axis.letter">
              <code-btn
                :color="axis.homed ? 'primary' : 'warning'"
                :disabled="uiFrozen"
                :title="$t('button.home.title', [axis.letter])"
                :code="`G28 ${axis.letter}`"
                block
              >{{ $t('button.home.caption', [axis.letter]) }}</code-btn>
            </v-flex>
          </v-layout>
          <v-layout justify-center row wrap class="hidden-md-and-up">
            <v-flex shrink class="px-1"></v-flex>
            <v-flex v-for="axis in displayedAxes" :key="axis.letter">
              <code-btn
                :color="axis.homed ? 'primary' : 'warning'"
                :disabled="uiFrozen"
                :title="$t('button.work.title', [axis.letter])"
                :code="`G10  L20 ${axis.letter}0\nM500`"
                block
              >{{ $t('button.work.caption', [axis.letter]) }}</code-btn>
            </v-flex>
          </v-layout>
        </v-layout>
      </v-layout>



      <v-layout row>
        <!-- Regular home buttons -->
        <v-flex shrink class="hidden-sm-and-down">
          <v-layout column>
            <v-flex>
              <code-btn
                block
                color="primary"
                code="G28"
                :title="$t('button.home.titleAll')"
                class="ml-0 hidden-sm-and-down"
              >{{ $t('button.home.captionAll') }}</code-btn>
            </v-flex>
           <v-flex shrink class="hidden-sm-and-down">
					<v-layout column>
						<v-flex v-for="axis in displayedAxes" :key="axis.letter">
							<code-btn block :color="axis.homed ? 'primary' : 'warning'" :disabled="uiFrozen" :title="$t('button.home.title', [axis.letter])" :code="`G28 ${axis.letter}`" class="ml-0">

								{{ $t('button.home.caption', [axis.letter]) }}
							</code-btn>
						</v-flex>
					</v-layout>
				</v-flex>
          </v-layout>
        </v-flex>

        <v-flex shrink class="hidden-sm-and-down px-1"></v-flex>

        <!-- Jog control -->
        <v-flex grow>
          <v-layout column>
            <v-flex grow class="hidden-sm-and-down">
              <v-menu full-width offset-y :disabled="uiFrozen" v-tab-control>
                <template slot="activator">
                  <v-btn block color="primary" class="mx-0" :disabled="uiFrozen">
                    {{ $t('panel.movement.compensation') }}
                    <v-icon>arrow_drop_down</v-icon>
                  </v-btn>
                </template>

                <v-card>
                  <v-list>
                    <template v-if="move.compensation">
                      <v-list-tile
                        class="center-menu-item"
                      >{{ $t('panel.movement.compensationInUse', [move.compensation]) }}</v-list-tile>

                      <v-divider></v-divider>
                    </template>

                    <v-list-tile @click="sendCode('G32')">
                      <v-icon class="mr-1">view_module</v-icon>
                      {{ $t(move.geometry.type === 'delta' ? 'panel.movement.runDelta' : 'panel.movement.runBed') }}
                    </v-list-tile>
                    <v-list-tile
                      :disabled="!move.compensation || move.compensation.indexOf('Point') === -1"
                      @click="sendCode('M561')"
                    >
                      <v-icon class="mr-1">clear</v-icon>
                      {{ $t('panel.movement.disableBedCompensation') }}
                    </v-list-tile>

                    <v-divider></v-divider>

                    <v-list-tile @click="sendCode('G29')">
                      <v-icon class="mr-1">grid_on</v-icon>
                      {{ $t('panel.movement.runMesh') }}
                    </v-list-tile>
                    <v-list-tile @click="showMeshEditDialog = true">
                      <v-icon class="mr-1">edit</v-icon>
                      {{ $t('panel.movement.editMesh') }}
                    </v-list-tile>
                    <v-list-tile @click="sendCode('G29 S1')">
                      <v-icon class="mr-1">save</v-icon>
                      {{ $t('panel.movement.loadMesh') }}
                    </v-list-tile>
                    <v-list-tile
                      :disabled="move.compensation !== 'Mesh'"
                      @click="sendCode('G29 S2')"
                    >
                      <v-icon class="mr-1">grid_off</v-icon>
                      {{ $t('panel.movement.disableMeshCompensation') }}
                    </v-list-tile>
                  </v-list>
                </v-card>
              </v-menu>
            </v-flex>
            <v-layout row wrap>
              <!-- Decreasing movements -->
              <v-flex>
                <v-layout row>
                  <!-- Decreasing movements -->
                  <v-flex
                    v-for="index in numMoveSteps"
                    :key="-index"
                    :class="getMoveCellClass(index - 1)"
                  >
                    <v-layout column>
                      <v-flex v-for="axis in displayedAxes" :key="axis.letter" class="mr-2">
                        <code-btn
                          :code="`M120\nG91\nG1 ${axis.letter}${-moveSteps(axis.letter)[index - 1]} F${Math.round(moveFeedrate * 60)}\nG90\nM121`"
                          no-wait
                          @contextmenu.prevent="showMoveStepDialog(axis.letter, index - 1)"
                          block
                          class="move-btn"
                        >
                          <v-icon>keyboard_arrow_left</v-icon>
                          {{ axis.letter + -moveSteps(axis.letter)[index - 1] }}
                        </code-btn>
                      </v-flex>
                    </v-layout>
                  </v-flex>

                </v-layout>
              </v-flex>

              <v-flex shrink class="hidden-sm-and-down px-1"></v-flex>

              <!-- Increasing movements -->
              <v-flex>
                <v-layout row>
                  <v-flex
                    v-for="index in numMoveSteps"
                    :key="index"
                    :class="getMoveCellClass(numMoveSteps - index)"
                  >
                    <v-layout column>
                      <v-flex v-for="axis in displayedAxes" :key="axis.letter" class="ml-2">
                        <code-btn
                          :code="`M120\nG91\nG1 ${axis.letter}${moveSteps(axis.letter)[numMoveSteps - index]} F${Math.round(moveFeedrate * 60)}\nG90\nM121`"
                          no-wait
                          @contextmenu.prevent="showMoveStepDialog(axis.letter, numMoveSteps - index)"
                          block
                          class="move-btn"
                        >
                          {{ axis.letter + '+' + moveSteps(axis.letter)[numMoveSteps - index] }}
                          <v-icon>keyboard_arrow_right</v-icon>
                        </code-btn>
                      </v-flex>
                    </v-layout>
                  </v-flex>
                </v-layout>
              </v-flex>
            </v-layout>
             <v-layout row wrap>
            <v-flex class="hidden-md-and-up">
          <code-btn
            block
            color="primary"
            :code="`G53 G0 Z94\nG1 X0 Y0 F1500\nG1 Z0 F1500`"
            :title="$t('button.workGoto.titleAll')"
          >{{ $t('button.workGoto.captionAll') }}</code-btn>
        </v-flex>
        </v-layout>
          </v-layout>
        </v-flex>


        <v-flex shrink class="hidden-sm-and-down px-1"></v-flex>



        <!-- Regular Set Work buttons -->
        <v-flex shrink class="hidden-sm-and-down">
          <v-layout column>
            <v-flex>
              <code-btn
                block
                color="primary"
                :code="`G10 L20 X0 Y0 Z0\nM500`"
                :title="$t('button.work.titleAll')"
                class="mr-0 hidden-sm-and-down"
              >{{ $t('button.work.captionAll') }}</code-btn>
            </v-flex>
            <v-flex grow v-for="axis in displayedAxes" :key="axis.letter">
              <code-btn
                block
                :color="axis.homed ? 'primary' : 'warning'"
                :disabled="uiFrozen"
                :title="$t('button.work.title', [axis.letter])"
                :code="`G10  L20 ${axis.letter}0\nM500`"
                class="mr-0"
              >{{ $t('button.work.caption', [axis.letter]) }}</code-btn>
            </v-flex>
          </v-layout>
        </v-flex>
      </v-layout>

       <!-- Regular Goto Work buttons -->
      <v-layout row>
        <v-flex>
          <code-btn
            block
            color="primary"
            :code="`G53 G0 Z94\nG1 X0 Y0 F1500\nG1 Z0 F1500`"
            :title="$t('button.workGoto.titleAll')"
            class="hidden-sm-and-down"
          >{{ $t('button.workGoto.captionAll') }}</code-btn>
        </v-flex>
        </v-layout>
    </v-card-text>

    <mesh-edit-dialog :shown.sync="showMeshEditDialog"></mesh-edit-dialog>
    <input-dialog
      :shown.sync="moveStepDialog.shown"
      :title="$t('dialog.changeMoveStep.title')"
      :prompt="$t('dialog.changeMoveStep.prompt')"
      :preset="moveStepDialog.preset"
      is-numeric-value
      @confirmed="moveStepDialogConfirmed"
    ></input-dialog>

    <v-alert v-if="unhomedAxes.length" :value="true" type="warning">
      {{ $tc('panel.movement.axesNotHomed', unhomedAxes.length) }}
      <strong>{{ unhomedAxes.map(axis => axis.letter).reduce((a, b) => `${a}, ${b}`) }}</strong>
    </v-alert>

    <v-alert :value="!move.axes.length" type="info">{{ $t('panel.movement.noAxes') }}</v-alert>
  </v-card>
</template>

<script>
"use strict";

import { mapState, mapGetters, mapActions, mapMutations } from "vuex";

export default {
  computed: {
    ...mapGetters(["isConnected", "uiFrozen"]),
    ...mapState("machine/model", ["move"]),
    ...mapState("machine/settings", ["moveFeedrate"]),
    ...mapGetters("machine/settings", ["moveSteps", "numMoveSteps"]),
    displayedAxes() {
      return this.move.axes.filter(axis => axis.visible);
    },
    unhomedAxes() {
      return this.move.axes.filter(axis => axis.visible && !axis.homed);
    }
  },
  data() {
    return {
      dropdownShown: false,
      showMeshEditDialog: false,
      moveStepDialog: {
        shown: false,
        axis: "X",
        index: 0,
        preset: 0
      }
    };
  },
  methods: {
    ...mapActions("machine", ["sendCode"]),
    ...mapMutations("machine/settings", ["setMoveStep"]),
    getMoveCellClass(index) {
      let classes = "";
      if (index === 0 || index === 5) {
        classes += "hidden-lg-and-down";
      }
      if (index > 1 && index < 4 && index % 2 === 1) {
        classes += "hidden-md-and-down";
      }
      return classes;
    },
    showMoveStepDialog(axis, index) {
      this.moveStepDialog.axis = axis;
      this.moveStepDialog.index = index;
      this.moveStepDialog.preset = this.moveSteps(this.moveStepDialog.axis)[
        this.moveStepDialog.index
      ];
      this.moveStepDialog.shown = true;
    },
    moveStepDialogConfirmed(value) {
      this.setMoveStep({
        axis: this.moveStepDialog.axis,
        index: this.moveStepDialog.index,
        value
      });
    }
  },
  watch: {
    isConnected() {
      // Hide dialogs when the connection is interrupted
      this.showMeshEditDialog = false;
      this.moveStepDialog.shown = false;
    }
  }
};
</script>
