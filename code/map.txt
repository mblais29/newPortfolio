<cfscript>
			
	buildingosis_qry = "";
	buildingosis_obj = createObject("component", "#request.osis_comproot#cfc.database.db_o_afm");
	buildingosis_obj.addTable("bl");
	buildingosis_obj.addField("bl_id");
	buildingosis_obj.addField("name");
	buildingosis_obj.addField("lat");
	buildingosis_obj.addField("lon");
	buildingosis_obj.addField("image_file");
	buildingosis_obj.addField("area_avg_em");
    buildingosis_obj.addField("area_rentable");
    buildingosis_obj.addField("count_em");
    buildingosis_obj.addField("count_max_occup");
	buildingosis_obj.addRestriction("lat","[[NOTNULL]]","EQ");
	buildingosis_obj.addRestriction("lon","[[NOTNULL]]","EQ");
	//dataosis_obj.addSecurity();
	buildingosis_qry = buildingosis_obj.runQuery();
	
	function getMapQuestKey() {

        var mapquestkey_qry = "";
        var ReturnValue = "";
        mapquestkey_obj = createObject("component", "#request.osis_comproot#cfc.database.db_osis_meta");
        mapquestkey_obj.addTable("osis_options");
        mapquestkey_obj.useCache(true);
        mapquestkey_obj.addField("option_val");
        mapquestkey_obj.addRestriction("var_name","o_mapping_mapquest_key","EQ");
        mapquestkey_qry = mapquestkey_obj.runQuery();

        if (mapquestkey_qry.recordcount EQ 1) {
            ReturnValue = mapquestkey_qry.option_val;
        }

        return ReturnValue;

    }
</cfscript>

<cfoutput>
    <script src="//open.mapquestapi.com/sdk/leaflet/v1.s/mq-map.js?key=#getMapQuestKey()#"></script>
</cfoutput>

<style>
	button.o_map_popup_button {
		padding: 5px;
		width: 130px;
		height: 30px;	
	}
	.leaflet-control-osis {
		box-shadow: 0 1px 7px rgba(0,0,0,0.4);
		background: #f8f8f9;
		-webkit-border-radius: 5px;
		border-radius: 5px;
		padding: 5px;
	}

    dd {
        margin-left: 10px;
    }
    dt {
        font-weight: 600;
    }
    dl {
        margin-top: 0;
    }
</style>

<script type='text/javascript'>
	
	var mapLayer = MQ.mapLayer();
    var map = new L.Map('map', {layers: mapLayer, center: new L.LatLng(42.737, -90.923), zoom: 4, zoomControl: true, detectRetina: true});

    L.control.layers({
        'Map': mapLayer,
        'Satellite': MQ.satelliteLayer(),
        'Hybrid': MQ.hybridLayer()
    }).addTo(map);
	
	var myIcon = L.icon({
	    iconUrl: 'tpl/o_common/o_leaflet/dist/images/comment-map-icon.png',
	    iconRetinaUrl: 'tpl/o_common/o_leaflet/dist/images/comment-map-icon-2x.png',
	    iconSize: [16, 19],
	    iconAnchor: [8,17],
	    labelAnchor: [2, -10]
	});
	
	var myAddIcon = L.icon({
	    iconUrl: 'tpl/o_common/o_leaflet/dist/images/comment-map-icon-add.png',
	    iconRetinaUrl: 'tpl/o_common/o_leaflet/dist/images/comment-map-icon-add-2x.png',
	    iconSize: [16, 19],
	    iconAnchor: [8,17],
	    labelAnchor: [2, -10]
	});
	
	var myIconAlt = L.AwesomeMarkers.icon({
	  icon: 'comments-o',
	  prefix: 'fa',
	  markerColor: 'green'
	});
	
	var myAddIconAlt = L.AwesomeMarkers.icon({
	  icon: 'comments-o',
	  prefix: 'fa',
	  markerColor: 'cadetblue'
	});
	
	<!---
	var map = new L.Map('map', {center: new L.LatLng(35, 0), zoom: 2, maxZoom: 15,  zoomControl: true, detectRetina: true});
	// add in Google layers
	L.tileLayer('http://{s}.tile.osm.org/{z}/{x}/{y}.png', {
    attribution: '&copy; <a href="http://osm.org/copyright">OpenStreetMap</a> contributors'
}).addTo(map);
	--->
	var markers = new L.MarkerClusterGroup({ spiderfyOnMaxZoom: true, showCoverageOnHover: true, zoomToBoundsOnClick: true, singleMarkerMode: false });
	
	<cfoutput query="buildingosis_qry">
		<cfset BuildingImageString = "" />
		<!---
		<cfif Len(image_file)>
			<cfset BuildingImageString = "<img style=""float:right; padding-left: 10px; padding-bottom: 10px"" width=""100"" src=""reports/rpt_image.cfm?img=#image_file#"">" />			
		</cfif>
		--->
		
		<cfquery name="vacant_workspace" datasource="osis_elpmain">
			SELECT count(*) AS vac_workspace FROM elpmain.rm WHERE rm.rm_cat IN ('OFFICE', 'WORKSTATION') AND rm.bl_id = <cfqueryparam value="#bl_id#" CFSQLType="CF_SQL_VARCHAR"/>
			and NOT EXISTS (SELECT 1 FROM elpmain.osis_pod_em_v WHERE osis_pod_em_v.bl_id=rm.bl_id AND osis_pod_em_v.fl_id=rm.fl_id AND osis_pod_em_v.rm_id=rm.rm_id AND osis_pod_em_v.status = 'ACTIVE')
		</cfquery>
		
		<cfquery name="headcountQry" datasource="osis_elpmain">
			SELECT COUNT(1) as headcount FROM elpmain.osis_pod_em_v WHERE osis_pod_em_v.bl_id= <cfqueryparam value="#bl_id#" CFSQLType="CF_SQL_VARCHAR"/> AND osis_pod_em_v.status = 'ACTIVE'
		</cfquery>
		
        /* add building marker(s) to map */
		var marker = new L.marker([#lat#, #lon#]);
		marker.bindPopup('<p style="width: 250px"><strong>#name# ( #bl_id# )</strong></p>#BuildingImageString#'
            + '<div style="float:left; width: 190px"><dl><dt>Rentable Area: </dt><dd>#area_rentable# sq. ft.</dd><dt>Headcount: </dt><dd>#headcountQry.headcount#</dd>'
            + '<dt>Vacant Workspaces: </dt><dd>#vacant_workspace.vac_workspace#</dd>'
            + '<dt>Rentable Area Per Employee: </dt><dd><cfif count_em neq 0>#NumberFormat(area_rentable / headcountQry.headcount, '9.99')#<cfelse>#area_rentable#</cfif> sq. ft.</dd>'
            + '</dl></div>'
            + '<iframe width="125" height="10" scrolling="no" style="border: none"></iframe>'
        );
		markers.addLayer(marker);
		
	</cfoutput>
	
	map.addLayer(markers);
	
	// add a small scale to lower left corner of map
	L.control.scale().addTo(map);
	
	// add search provider
	/* 
	new L.Control.GeoSearch({        
		provider: new L.GeoSearch.Provider.Google()
    }).addTo(map);
    */
    
    new L.Control.GeoSearch({
        provider: new L.GeoSearch.Provider.OpenStreetMap()
    }).addTo(map);
	
	// add layers control
	//map.addControl(new L.Control.Layers({'Google - Street':googleLayerRoadMap, 'Google - Terrain':googleLayerTerrain, 'Google - Hybrid':googleLayerHybrid},{}));

	var drawnItems = new L.FeatureGroup(); 
	map.addLayer(drawnItems);  
	
	var drawControl = new L.Control.Draw({ draw: { position: 'topleft', 
		polygon: { title: 'Draw a polygon!', allowIntersection: false, drawError: { color: '#662d91', timeout: 1000 }, shapeOptions: { color: '#662d91' }, showArea: true }, 
		polyline: { metric: false }, 
		circle: { shapeOptions: { color: '#662d91' } } }, 
		edit: { featureGroup: drawnItems } }); 
	
	map.addControl(drawControl);  
	
	map.on('draw:created', function (e) { var type = e.layerType, layer = e.layer;  if (type === 'marker') { layer.bindPopup('A popup!'); }  drawnItems.addLayer(layer); }); 
		
		
	var osm = new L.FeatureGroup(); 
	map.addLayer(osm);
	
	L.Control.measureControl().addTo(map);
	
	
	
	function generateUUID() {
	    var d = new Date().getTime();
	    var uuid = 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function(c) {
	        var r = (d + Math.random()*16)%16 | 0;
	        d = Math.floor(d/16);
	        return (c=='x' ? r : (r&0x7|0x8)).toString(16);
	    });
	    return uuid;
	};
	
	function showLink(LinkURL) {
	
		if (parent.parent.document.getElementById("info")) {
			parent.parent.document.getElementById("info").src = LinkURL;
		} else {
			showInPopUp(LinkURL);
		}
	
	}
	
	function showInPopUp(ShowURL) {

		document.getElementById("o_map_popframe_id").src = ShowURL;
		$( "#o_map_dialog_id" ).dialog( "open" );
	
	}


</script>