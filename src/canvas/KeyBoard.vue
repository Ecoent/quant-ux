<script>

import on from 'dojo/on'
import css from 'dojo/css'
import lang from 'dojo/_base/lang'
import keys from 'dojo/keys'
import topic from 'dojo/topic'
import Dialog from 'common/Dialog'
import win from 'dojo/win'

export default {
    name: 'KwyBoard',
    mixins:[],
    data: function () {

      return {
      }
    },
    components: {},
    methods: {

      initKeys () {
        this.own(on(win.body(), "keydown", lang.hitch(this,"onKeyPress")));
			  this.own(on(win.body(), "keyup", lang.hitch(this,"onKeyUp")));
      },

      onKeyPress (e){

        this._currentKeyEvent = e;
        var k = e.keyCode ? e.keyCode : e.which;
        var target = e.target;
        var isMeta = e.altKey || e.ctrlKey || e.metaKey;
        var isCntrl = e.ctrlKey || e.metaKey;
        var isShift = e.shiftKey



        // console.debug("onKeyPress", target, isMeta, css.contains(target, "MatcIgnoreOnKeyPress"))

        /**
         * Cancel listeners must be always fired.
         */
        if (k == keys.ESCAPE){
          this.onCancelAction();
          topic.publish("matc/canvas/esc");
          this.stopEvent(e);
          return
        }

        /**
         * IF we have a dialog open, we return
         */
        if (this.state == "simulate" || this.state == "dialog" || Dialog.getCurrentDialog()) {
          this.logger.log(1 ,"onKeyPress", "exit because of dialog");
          return;
        }

        /**
         * Inputs from the toolbar should be also ignored
         */
        if (css.contains(target, "MatcIgnoreOnKeyPress")){
          return;
        }

        this._currentKeyPressed = k;

        /**
         * Do nothing if we are in inline edit
         */
        if (this._inlineEditStarted ){
          this.onSelectionKeyPress(e);
        /**
         * Arrow dispatch if cntrl is not pressed
         */
        } else if(k == 37 && !isCntrl){
          if(!this._inlineEditStarted ){
            this.onArrowLeft(e, isShift);
            this.stopEvent(e);
          }
        } else if(k == 39 && !isCntrl){
          if(!this._inlineEditStarted){
            this.onArrowRight(e, isShift);
            this.stopEvent(e);
          }
        } else if(k == 40 && !isCntrl){
          if(!this._inlineEditStarted){
            this.onArrowDown(e, isShift);
            this.stopEvent(e);
          }
        } else if(k == 38 && !isCntrl){
          if(!this._inlineEditStarted ){
            this.onArrowUp(e, isShift);
            this.stopEvent(e);
          }
        } else if (k == 65) { // a for select

          if(!this._inlineEditStarted && !this._resizeStartPos && !this._selectionToolStart){
            this.setMode("select");
            /**
             * Start selection tool
             */
            this.unSelect();
            this._selectionToolStart = this._lastMousePos;
            this._selectionToolMoveListener = on(win.body(),"mousemove", lang.hitch(this,"onSelectionMove"));
            this.showHint("Move mouse to start selecting...");
          }
        } else if (k==18){ // alt
          if(!this._inlineEditStarted && !this._resizeStartPos && !this._dragNDropBoxWidgetStart){
            if(this.mode == "edit"){
              this.setMode("distance");
              if(this._selectWidget){
                this.renderScreenDistance();
              }	else {
                if(this._lastHoverWidget){
                  this.renderWidgetDistance(this._lastHoverWidget);
                }
              }
            } else {
              console.debug("ALT while", this._dragNDropBoxWidgetStart);
            }
          }
        } else if (k==32){ // space

          if(!this._inlineEditStarted ){
            this.stopEvent(e);
            if(this.getMode() != "move"){
              this.showHint("Move the mouse to move canvas...");
              this.onDragStart(this.container, "container", "onCanvasDnDStart", "onCanvasDnDMove", "onCanvasDnDEnd", null, this._lastMouseMoveEvent, true);
              /**
               * If we are adding a line, we do not want to change mode ( and trigger redraw).
               * Instead we block the Add._updateAddLineMove() method by setting the pause flag.
               */
              if (this.getMode() != "addLine") {
                this.setMode("move");
                this.setDnDMinTime(0);
              } else {
                this._addLineIsPaused = true;
              }
            }
          }

        /**
         * H dispatch...
         */
        } else if(k == 72){
          if(!this._inlineEditStarted && !this._selectionToolStart){
            this.setMode("hotspot");
            this.unSelect();
            this.showHint("Mark the area where to create the hotspot...");
            this.stopEvent(e);
          }
        /**
         * R dispatch...
         */
        } else if(k == 82){
          if(!this._inlineEditStarted  && !this._selectionToolStart){
            this.setMode("addBox");
            this.unSelect();
            this.showHint("Mark the area where to create the box...");
            this.stopEvent(e);

          }
        /**
         * C dispatch...
         */
        } else if(k == 67 && !isMeta){
          if(!this._inlineEditStarted  && !this._selectionToolStart){
            if (this._selectWidget || this._selectGroup || this._selectMulti) { // _selectGroup
              this.onReplicate();
              this.stopEvent(e);
              e.cancelBubble = true
            } else {
              this.showHint("Cloning does not work on multi selection");
            }
          }
        /**
         * L dispatch...
         */
        } else if(k == 76){
          if(!this._inlineEditStarted  && !this._selectionToolStart){
            this.addLineAtSelected(e)
          }
        /**
         * D dispatch...
         */
        } else if(k == 68 && !isMeta){
          if(!this._inlineEditStarted  && !this._selectionToolStart){
            if (this._selectMulti) {
              this.onDistribute();
              this.stopEvent(e);
              e.cancelBubble = true
            } else {
              this.showHint("Select multiple widgets to distribute equally...");
            }
          }
        /**
         * T dispatch...
         */
        } else if(k == 84){
          if(!this._inlineEditStarted  && !this._selectionToolStart){
            this.setMode("addText");
            this.showHint("Mark the area where to create the txt...");
            this.stopEvent(e);
          }
        /**
         * W dispatch...
         */
        } else if(k == 87){
          if(!this._inlineEditStarted){
            if(this.toolbar){
              this.toolbar.showWidgetSelector();
            }
          }
        /**
         * S dispatch...
         */
        } else if(k == 83){
          if(!this._inlineEditStarted){
            if(this.toolbar){
              this.toolbar.showScreenSelector();
            }
          }

        /**
         * Zoom
         */
        } else if (k== 171 || k ==187){ // +

          if(!this._inlineEditStarted){
            this.onClickPlus();
            this.stopEvent(e);
          }
        } else if (k== 173 || k ==189){ //-

          if(!this._inlineEditStarted){
            this.onClickMinus();
            this.stopEvent(e);
          }
        } else if(k == keys.DELETE || k == keys.BACKSPACE){
          var removed = this.onRemoveSelected();
          if(removed){
            this.stopEvent(e);
          }
        } else if (e.altKey || e.ctrlKey || e.metaKey){

          this.logger.log(1,"onKeyPress", "enter > " + k + " > ctrl : " +e.ctrlKey + " > meta :" +(e.ctrlKey || e.metaKey));

          /**
           * Copy only when no inline edit
           */
          if(!this._inlineEditStarted){
            if(k == 67){ // ctrl-c
              this.onCopy();
              this.stopEvent(e);
            }
            if(k == 86){ // ctrl -v
              this.onPaste();
              this.stopEvent(e);
            }
            if(k == 88){// ctrl-x
              this.onCut();
              this.stopEvent(e);
            }
            if(k == 90){// ctrl-z
              this.controller.undo();
              this.stopEvent(e);
            }
            if(k == 89){// ctrl-y
              this.controller.redo();
              this.stopEvent(e);
            }

            if(k == 68){ // ctrl-d
              this.onDuplicate();
              this.stopEvent(e);
            }

            if(k == 40){ // ctrl & down
              if(this.toolbar){
                this.stopEvent(e);
                this.toolbar.onToolWidgetLayer("back");
              }
            }

            if(k == 38){ // ctrl + up
              if(this.toolbar){
                this.stopEvent(e);
                this.toolbar.onToolWidgetLayer("front");
              }
            }

            if(k == 71){ // ctrl-g
              this.onGroup();
              this.stopEvent(e);
            }
          }

        } else {
          /**
           * Default like inline edit
           */
          this.onSelectionKeyPress(e);
        }

      },

      getCurrentKeyCode  () {
        if(this._currentKeyEvent){
          return this._currentKeyEvent.keyCode ? this._currentKeyEvent.keyCode : this._currentKeyEvent.which
        }
        return -1;
      },

      onKeyUp (e){

        if(this.state == "simulate" || this.state == "dialog" || Dialog.getCurrentDialog()){
          this.logger.log(1 ,"onKeyUp", "exit because of dialog");
          return;
        }

        var target = e.target;
        if(css.contains(target, "MatcIgnoreOnKeyPress")){
          return
        }

        var k = e.keyCode ? e.keyCode : e.which;

        if(this._inlineEditStarted ){
          /**
           * Do nothing...
           */
            return;
        } else if (k==65){
          /**
           * End selection
           */
          this.onSelectionEnd();
          this.setMode("edit");
        } else if (k==18){ // alt
          this.cleanUpAlignment();
          this.setMode("edit");
        } else if (k==32){ // space
          this.onDragEnd(this._lastMouseMoveEvent);
          /**
           * Enable line Add._updateAddLineMove again.
           * Set mode to edit, if we are not adding a line
           */
          this._addLineIsPaused = false;
          if (this.getMode() != "addLine") {
            this.setMode("edit");
          }
          this.stopEvent(e);
        } else if (k==84){ // t

        }	else if (k==68){ // D

        } else if (k==72){ // H

        } else if (k==82){ // B

        } else if (k == 72 || k == 84 || k == 66 || k == 70){
          this.stopEvent(e);
          this.setMode("edit");
        }

        delete this._currentKeyPressed;
        delete this._currentKeyEvent;
      }
    }

}
</script>