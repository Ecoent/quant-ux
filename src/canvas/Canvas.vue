<template>
<div class="MatcCanvas">
	<div class="MatcCanvasFrame" data-dojo-attach-point="frame">
		<div class="MatcCanvasContainer MatcCanvasZoomable " data-dojo-attach-point="container">
			<div class="MatcCanvasContainer " data-dojo-attach-point="zoomContainer">
				<div data-dojo-attach-point="screenContainer" class="MatcCanvasLayer"></div>
				<div data-dojo-attach-point="widgetContainer" class="MatcCanvasLayer"></div>
			</div>
			<div data-dojo-attach-point="dndContainer" class="MatcDnDLayer"></div>
		</div>
	</div>
	<div class="MatcCanvasScrollBar MatcCanvasScrollBarRight" data-dojo-attach-point="scrollRight">
		<div class="MatcCanvasScrollBarCntr MatcCanvasScrollBarCntrRight" data-dojo-attach-point="scrollRightCntr">
			<div class="MatchCanvasScrollHandle" data-dojo-attach-point="scrollRightHandler"></div>
		</div>
	</div>
	<div class="MatcCanvasScrollBar MatcCanvasScrollBarBottom" data-dojo-attach-point="scrollBottom">
		<div class="MatcCanvasScrollBarCntr MatcCanvasScrollBarCntrBottom" data-dojo-attach-point="scrollBottomCntr">
			<div class="MatchCanvasScrollHandle" data-dojo-attach-point="scrollBottomHandler"></div>
		</div>
	</div>
	<!-- Status -->
	<div class="MatcMessage" data-dojo-attach-point="message">
	</div>
</div>

</template>
<script>
import DojoWidget from 'dojo/DojoWidget'
import css from 'dojo/css'
import Logger from 'common/Logger'
import on from 'dojo/on'
import touch from 'dojo/touch'
import lang from 'dojo/_base/lang'
import win from 'dojo/win'
import topic from 'dojo/topic'
import _DragNDrop from 'common/_DragNDrop'
import Dialog from 'common/Dialog'
import DomBuilder from 'common/DomBuilder'
import GridSelector from 'canvas/GridSelector'
import Render from 'canvas/Render'
import Lines from 'canvas/Lines'
import DnD from 'canvas/DnD'
import Add from 'canvas/Add'
import Select from 'canvas/Select'
import Distribute from 'canvas/Distribute'
import Tools from 'canvas/Tools'
import Zoom from 'canvas/Zoom'
import Util from 'core/Util'
import InlineEdit from 'canvas/InlineEdit'
import Scroll from 'canvas/Scroll'
import Upload from 'canvas/Upload'
import Comment from 'canvas/Comment'
import Layer from 'canvas/Layer'
import DataView from 'canvas/DataView'
import ScreenRuler from 'canvas/ScreenRuler'
import CustomHandler from 'canvas/CustomHandler'

import KeyBoard from 'canvas/KeyBoard'
import Resize from 'canvas/Resize'
import Replicate from 'canvas/Replicate'
import Prototyping from 'canvas/Prototyping'

import FastDomUtil from 'core/FastDomUtil'

export default {
  name: 'Canvas',
	mixins:[DojoWidget, _DragNDrop, Util, Render, Lines, DnD, Add, Select, Distribute, Tools,
			Zoom, InlineEdit, Scroll, Upload, Comment, Layer, CustomHandler, ScreenRuler, DataView,
			KeyBoard, Resize, Replicate, Prototyping],
    data: function () {
        return {
					mode: "edit",
					isMoveOnlySelected: true,
          debug: false,
          grid: null,
          isPublic: false,
					active: true,
					name: 'XCanvas'
        }
    },
    components: {},
    methods: {
		postCreate (){

			this.logger = new Logger("Canvas")
			this.logger.log(2,"postCreate", "entry");
			this.domUtil = new FastDomUtil()
			this.initSize()

			/**
			 * init container size and position
			 */
			this.canvasPos = {
				x : this.canvasStartX,
				y: this.canvasStartY,
				w: this.canvasFlowWidth,
				h: this.canvasFlowHeight
			};
			this.initContainerSize();
			this.setContainerPos();

			/**
			 * Init remaining sub components
			 */
			this.initSettings();
			this.initRender();
			this.initMouseTracker();
			this.initZoom();
			this.initScrollBars();
			this.initUpload();
			this.initComment();
			this.initScreenRuler()
			this.initDataView()
			this.initWiring()
			this.initKeys()

			/**
			 * Init Listeners
			 */
			this.own(topic.subscribe("matc/toolbar/click", lang.hitch(this,"onToolbarClick")));
			//this.own(on(this.gridBtn, touch.press, lang.hitch(this, "showGrid")));

			this.hasCustomHandler = false


			/**
			 * Set correct mode
			 */
			css.add(this.domNode, "MatcCanvasMode"+ this.mode);
			this.logger.log(-1,"postCreate", "exit > " + this.mode);
		},

		setUser (u){
			this.user = u;
		},

		setPublic (isPublic){
			this.isPublic = isPublic;
		},

		setModelService (s) {
			this.modelService = s
		},

		setCommentService (s) {
			this.commentService = s
		},

		setController (c){
			this.controller = c;
			c.setCanvas(this);
		},

		getController (){
			if(this._controllerCallback){
				this[this._controllerCallback]();
			}
			return this.controller;
		},

		setControllerCallback (c){
			this._controllerCallback = c;
		},

		setViewMode (m) {
			this.logger.log(-1, "setViewMode", "enter > " + m);
			this.canvasViewMode = m
			this.setPrototypingView(m === 'prototype')
		},

		setModelFactory (f){
			this.factory = f;
		},

		setRenderFactory (f){
			this.renderFactory = f;
		},

		setToolbar (t){
			this.toolbar = t;
			if (this.settings){
				this.toolbar.setSettings(this.settings);
			}
			this.onChangeCanvasViewConfig()
		},

		onChangeCanvasViewConfig () {
			if (this.toolbar) {
				let hasGrid = false
				let hasShowGrid = false
				let grid = null
				if (this.model) {
					hasGrid = this.model.grid.enabled
					hasShowGrid = this.model.grid.visible
					grid = this.model.grid
				}
				this.toolbar.setCanvasViewConfig({
					zoom: this.zoom,
					renderLines: this.renderLines,
					showDistance: this.showDistance,
					showComments:  this.showComments,
					showRuler: this.showRuler,
					hasDataView: this.hasDataView,
					layerListVisible: this.settings.layerListVisible,
					hasGrid: hasGrid,
					hasVisibleGrid: hasShowGrid,
					grid: grid
				})
			}
		},

		setCanvasViewConfig (key, value) {
			this.logger.log(-1, "setCanvasViewConfig", "enter > " + key, value);
			if (key === 'zoom') {
				this.setZoomFactor(value)
			}

			if (key === 'renderLines') {
				this.setViewLines(value)
			}

			if (key === 'showDistance') {
				this.setShowDistance(value)
			}

			if (key === 'showComments') {
				this.setCommentView(value)
			}

			if (key === 'showRuler') {
				this.setShowScreenRuler(value)
			}

			if (key === 'hasGrid') {
				this.setEnableGrid(value)
			}

			if (key === 'hasVisibleGrid') {
				this.setVisibleGrid(value)
			}

			if (key === 'showGrid') {
				this.showGrid(value)
			}

			if (key === 'layerListVisible') {
				this.setLayerVisibility(value)
			}

			if (key === 'hasDataView') {
				this.setDataView(value)
			}
		},

		setScreenName (screen) {
			this.logger.log(-1, "setScreenName", "enter > " + screen.name);
			if (this.layerList){
				this.layerList.changeName(screen);
			}
			/**
			 * FIXME: here is a small bug. After yooming I get in here still a reference to the old
			 * label. This the updates are not visible
			 */
			let div = this.screenLabels[screen.id]
			if (div) {
				this.setTextContent(div, screen.name);
			} else {
				this.logger.log(-1, "setScreenName", "exit  > NO LABEL" + screen.name, this.screenLabels);
			}
		},

		setWidgetName (widget) {
			this.logger.log(-1, "setWidgetName", "enter > " + widget.name);
			if (this.layerList){
				this.layerList.changeName(widget);
				/**
				 * Also update inherited
				 */
				let viewWidget = this.model.widgets[widget.id]
				if (viewWidget && viewWidget.copies) {
					viewWidget.copies.forEach(copyId => {
						let copyWidget = this.model.widgets[copyId]
						if (copyWidget) {
							this.layerList.changeName(copyWidget);
						}
					})
				}
			}
		},

		setGroupName (group) {
			this.logger.log(-1, "setGroupName", "enter > " + group.name);
			if (this.layerList){
				this.layerList.changeName(group);
			}
		},

		setModel (model){
			this.model = model;
			this.onChangeCanvasViewConfig()
		},


		setMode (mode, forceRender){
			this.logger.log(3,"setMode", "enter > " + mode +" != " + this.mode + " > forceRender : " + forceRender);
			if(mode != this.mode ){

				/**
				 * Toggle mode specify css class
				 */
				css.remove(this.domNode, "MatcCanvasMode"+this.mode);
				css.add(this.domNode, "MatcCanvasMode"+mode);

				this.mode = mode;
				if(this.toolbar){
					this.toolbar.setMode(mode);
				}
				/**
				 * FIXME: Make sure this is correct!
				 */
				//this.rerender();
				if(forceRender){
					this.rerender();
				} else {
					this.inlineEditStop();
					this.renderSelection();
				}

			} else if(forceRender){
				this.rerender();
			}
		},

		getMode (){
			return this.mode;
		},

		getStatusBar (){
			return this.status;
		},

		onExit (){
			this.logger.log(-1,"onExit", "enter > " );
			this.active = false;
		},



		/***************************************************************************
		 * Settings
		 ***************************************************************************/

		initSettings (){
			this.logger.log(0, "initSettings", "enter > " );
			/**
			 * default settings.
			 * Since 3.0.43 we snapp by default to top left corner
			 */
			this.settings = {
				canvasTheme : "MatcLight",
				lineColor : "#3787f2",
				lineWidth : 1,
				storePropView : true,
				moveMode : "ps",
				startToolsOnKeyDown : true,
				mouseWheelMode : "scroll",
				renderLines : false,
				snapGridOnlyToTopLeft: true,
				keepColorWidgetOpen: true,
				layerListVisible: false,
				showRuler: true,
				fastRender: false,
				hasProtoMoto: false,
				zoomSnapp: true,
				selectMove: true,
				hasDesignToken: true
			};

			var s = this._getStatus("matcSettings");
			if (s){
				this.mergeSettings(s)
			} else {
				this.logger.log(2,"initSettings", "exit>  no saved settings" );
			}
			this.applySettings(this.settings);

		},

		getSettings (){
			return this.settings;
		},

		mergeSettings (s) {
			if(s.canvasTheme){
				this.settings.canvasTheme = s.canvasTheme;
			}
			if(s.lineColor){
				this.settings.lineColor = s.lineColor;
			}
			if(s.lineWidth){
				this.settings.lineWidth = s.lineWidth;
			}
			if(s.moveMode){
				this.settings.moveMode = s.moveMode;
			}
			if(s.storePropView!=null){
				this.settings.storePropView = s.storePropView;
			}
			if(s.startToolsOnKeyDown!=null){
				this.settings.startToolsOnKeyDown = s.startToolsOnKeyDown;
			}
			if(s.mouseWheelMode!=null){
				this.settings.mouseWheelMode = s.mouseWheelMode;
			}
			if(s.keepColorWidgetOpen === true || s.keepColorWidgetOpen === false){
				this.settings.keepColorWidgetOpen = s.keepColorWidgetOpen;
			}
			if (s.layerListVisible === true || s.layerListVisible === false){
				this.settings.layerListVisible = s.layerListVisible;
			}
			if (s.fastRender != null) {
				this.settings.fastRender = s.fastRender
			}
			if (s.snapGridOnlyToTopLeft != null) {
				this.settings.snapGridOnlyToTopLeft = s.snapGridOnlyToTopLeft
			}
			if (s.hasProtoMoto != null) {
				this.settings.hasProtoMoto = s.hasProtoMoto
			}
			if(s.zoomSnapp === true || s.zoomSnapp === false){
				this.settings.zoomSnapp = s.zoomSnapp
			}
			if (s.selectMove === true || s.selectMove === false) {
				this.settings.selectMove = s.selectMove
			}
			if (s.hasDesignToken === true || s.hasDesignToken === false) {
				this.settings.hasDesignToken = s.hasDesignToken
			}
		},

		/**
		 * Called from the dialog
		 */
		setSettings (s){
			this.mergeSettings(s)
			this._setStatus("matcSettings",this.settings);
			this.applySettings(this.settings);
			this.rerender();
		},

		applySettings (s){
			this.logger.log(0,"applySettings", "enter > "  + s.canvasTheme + " &> " + s.moveMode);

			if(s.moveMode){
				this.moveMode = s.moveMode;
			}

			/**
			 * Since 4.0.0 we ignore the setting. Only the EditModeButton determines line visiboility
			 */
			//if(s.renderLines!=null){
				//this.renderLines = s.renderLines;
			//}

			if(s.showDistance!=null){
				this.showDistance = s.showDistance;
			}

			if(s.showRuler!=null){
				this.showRuler = s.showRuler;
			}

			if(s.showAnimation!=null){
				this.showAnimation = s.showAnimation;
			}

			if(s.lineColor){
				this.defaultLineColor = s.lineColor;
			}

			if(s.lineWidth){
				this.defaultLineWidth = s.lineWidth;
			}

			if (s.mouseWheelMode){
				this._mouseWheelMode = s.mouseWheelMode;
			}

			if (s.zoomSnapp === false  || s.zoomSnapp === true) {
				this.setSettingZoomSnap(s.zoomSnapp)
			}

			if (s.selectMove === true || s.selectMove === false) {
				this.isMoveOnlySelected = s.selectMove
			}

			if (s.canvasTheme){
				if(this._lastCanvasTheme){
					css.remove(win.body(), this._lastCanvasTheme);
				}
				css.add(win.body(), s.canvasTheme)
				this._lastCanvasTheme = s.canvasTheme;

				/**
				 * FIXME: Kind of hack
				 */
				if(s.canvasTheme=="MatcLight"){
					this.defaultLineColor = "#49C0F0";
				} else {
					this.defaultLineColor = "#49C0F0";
				}
			}

			this.settings = s;

			if (this.toolbar) {
				this.toolbar.setSettings(this.settings);
			}
			this.onChangeCanvasViewConfig()
			//console.debug("applySetztings() > exit > renderlines: ", this.renderLines, " > showSettings: ", this.showComments);
		},



		/***************************************************************************
		 * Grid
		 ***************************************************************************/


		showGrid (target){

			var db = new DomBuilder();
			var popup = db.div("MatcGridSelectorDialogContent MatcPadding").build();

			var selector = this.$new(GridSelector);
			selector.setValue(this.controller.model);
			selector.placeAt(popup)

			var dialog = this.createDialog();
			var bar = db.div("container").div("row").div("col-md-12").div("MatcButtonBar MatcMarginTop").build(popup);
			var write = db.div("MatcButton", "Save").build(bar);
			var cancel = db.a("MatcLinkButton ", "Cancel").build(bar);

			dialog.own(on(cancel, touch.press, lang.hitch(this, "closeDialog")));
			dialog.own(on(write, touch.press, lang.hitch(this, "setGrid2", selector)));
			if (target.screenX) {
				target = this.gridBtn
			}
			dialog.popup(popup, target);
		},

		setEnableGrid (value) {
			let grid = lang.clone(this.model.grid)
			grid.enabled = value
			if (grid.type === "columns"){
				this.controller.setGrid2(grid, "rgba(0,0,0,0.25)", "line");
			} else {
				this.controller.setGrid2(grid, "#cecece", "line");
			}
		},

		setVisibleGrid (value) {
			this.renderGridUpdates();
			let grid = lang.clone(this.model.grid)
			grid.visible = value
			if (grid.type === "columns"){
				this.controller.setGrid2(grid, "rgba(0,0,0,0.25)", "line");
			} else {
				this.controller.setGrid2(grid, "#cecece", "line");
			}
		},

		setGrid2 (selector){

			if(selector.isValid()){
				this.renderGridUpdates();
				var grid = selector.getValue();
				this.gridBackground = {}
				if (grid.type === "columns"){
					this.controller.setGrid2(grid, "rgba(0,0,0,0.25)", "line");
				} else {
					this.controller.setGrid2(grid, "#cecece", "line");
				}
				this.closeDialog();
			} else {
				this.dialog.shake();
			}
		},

		isInt (value){
			var er = /^-?[0-9]+$/;
			return er.test(value);
		},

		/***************************************************************************
		 * Dialog Handling
		 ***************************************************************************/

		createDialog (){
			this.dialog = new Dialog();
			//this.dialog.wrapperClass =  "MatcCanvasDialogWrapper";
			this.state = "dialog";
			return this.dialog;
		},

		closeDialog (){
			this.state =0;

			if(this.dialog){
				this.dialog.close();
			}
			this.dialog = null;
		},

		setState (s){
			this.state = s;
		},


		/***************************************************************************
		 * Helper Functons
		 ***************************************************************************/

		initMouseTracker (){
			this.own(on(win.body(),"mousemove", lang.hitch(this,"onMouseMove")));
		},

		onMouseMove (e){
			var pos2 = this.getCanvasMousePosition(e, true);
			/**
			 * if we show, only with in screen position... dunno if this gets to slow...
			 */
			this._lastMousePos = pos2;
			this._lastMouseMoveEvent = e;
		},
		destroy (){
			this.cleanUp();
			this._dojoCleanUpOwn();
			this._dojoCleanUpEvent();
			this.cleanUpTempListener();
		}
	},
    mounted () {
    }
}
</script>