<template>
    <lambda-form-item :label=label :name="model.component" :meta="meta">
        <div class="geographic">
            <div class="geographic-mapx"  :id="`geographic_${model.component}`" style="height: 100%; width: 100%; border: 1px solid #ccc;">



            </div>
            <div id="base-maps">
                <ul>
                    <li v-for="(baseMap, index) in baseMaps" :key="index">
                        <a href="javascript:;" @click="changeBaseMap(index)"
                           :class="index == currentBaseMap ? 'active' : ''">
                            {{baseMap.title}}
                        </a>
                    </li>
                </ul>
            </div>
            <div id="side_bar" :class="openSide ? 'open' : ''">
                <a-button @click="openSide = !openSide"
                        :class="openSide ? 'side-toggle show' : 'side-toggle'">
                    <template #icon>
                        <RightSquareOutlined v-if="openSide" />
                        <LeftSquareOutlined v-else />
                    </template>
                </a-button>


                <h3>{{lang.graphicsManagement}}</h3>
                <hr>


                <a-radio-group v-model:value="geometryType" class="geometry_type" v-if="allowMultiGeometryTypes">
                    <a-radio value="point" :disabled="current !== null">
                        <span>{{lang.point}}</span>
                    </a-radio>
                    <a-radio value="line" :disabled="current !== null">
                        <span>Line</span>
                    </a-radio>
                    <a-radio value="polygon" :disabled="current !== null">
                        <span>{{lang.polygon}}</span>
                    </a-radio>
                </a-radio-group>

                <hr>

                <table>
                    <thead>
                    <tr>
                        <th>{{lang.longitude}}</th>
                        <th>{{lang.latitude}}</th>
                    </tr>
                    </thead>
                    <tbody>

                    <tr v-for="(point, index) in points" :key="index">
                        <td>
                            <input type="text" v-model="point[0]"/>
                        </td>
                        <td>
                            <input type="text" v-model="point[1]"/>
                        </td>
                    </tr>
                    <tr>
                        <td>
                            <input type="text" ref="lng" v-model="new_point.lng" @change="createNewGeometry"/>
                        </td>
                        <td>
                            <input type="text" ref="lat" v-model="new_point.lat" @change="createNewGeometry"/>
                        </td>
                    </tr>
                    </tbody>
                </table>

                <a-button type="success" @click="saveGraphic" v-if="points.length >= 1">{{lang.save}}</a-button>
                <a-button type="warning" @click="cancelGraphic" v-if="points.length >= 1">{{lang.cancel}}</a-button>


                <!--<Button type="success" @click="getData">Get data</Button>-->


            </div>
            <!--<div id="spatial_query" v-if="featureLayerUrl" >-->
            <!--ТЗ-н тайлбайгаар шалгах  <i-switch size="small" v-model="checkByArea" @on-change="removeEditeLauyer"></i-switch>-->
            <!--<hr>-->
            <!--<div v-if="checkByArea">-->
            <!--<Form ref="searchForm" :model="formInline" :rules="ruleInline" inline>-->
            <!--<a-form-item prop="search_value">-->
            <!--<Input type="text" size="small" v-model="formInline.search_value"-->
            <!--@on-enter="handleSubmit"-->
            <!--:placeholder="searchFieldPlaceHolder"/>-->


            <!--</a-form-item>-->

            <!--<a-form-item>-->
            <!--<Button :disabled="searchLoading" :loading="searchLoading" @click="handleSubmit"-->
            <!--size="small" shape="circle" icon="ios-search-outline">-->

            <!--</Button>-->
            <!--</a-form-item>-->
            <!--</Form>-->
            <!--</div>-->


            <!--</div>-->
        </div>
    </lambda-form-item>
</template>
<script>
import axios from "axios"
import { LeftSquareOutlined, RightSquareOutlined } from '@ant-design/icons-vue';
import "leaflet";
import shadowUrl from  "leaflet/dist/images/marker-shadow.png";
import iconRetinaUrl from "leaflet/dist/images/marker-icon-2x.png";
import iconUrl from "leaflet/dist/images/marker-icon.png";

import "leaflet-draw";
const iconDefault = L.icon({
    iconRetinaUrl,
    iconUrl,
    shadowUrl,
    iconSize: [25, 41],
    iconAnchor: [12, 41],
    popupAnchor: [1, -34],
    tooltipAnchor: [16, -28],
    shadowSize: [41, 41]
});

import {point, polygon, inside} from '@turf/turf'
import mixin from './_mixin'
import {notification} from "ant-design-vue";

window.type = true;
export default {
    mixins: [mixin],

    computed: {
        lang() {
            const labels = ['save', 'cancel', 'longitude', 'latitude', 'graphicsManagement', 'polygon', 'point', 'noSiteFound',  'theSiteHasNotBeenSelected',

            ];
            return labels.reduce((obj, key, i) => {
                obj[key] = this.$t('dataForm.' + labels[i]);
                return obj;
            }, {});
        },
        options() {
            return this.meta.options;
        },
        value_field_form() {


            if (this.formValueField)
                return this.model.form[this.formValueField]
            else
                return undefined;
        }
    },
    components: {
        LeftSquareOutlined,
        RightSquareOutlined
    },
    mounted() {
        if (this.meta.GeographicOption) {
            this.zoom = this.meta.GeographicOption.zoom;
            this.attributes = this.meta.GeographicOption.attributes.split(',');
            this.currentBaseMap = this.meta.GeographicOption.baseMap;
            this.geometryType = this.meta.GeographicOption.geometryType;
            this.center = this.meta.GeographicOption.center;
            this.allowMultiGeometryTypes = false;
            this.allowMultiGeometries = false;
            this.checkByArea = this.meta.GeographicOption.checkByArea;
            this.featureLayerUrl = this.meta.GeographicOption.featureLayerUrl;
            this.featureTitles = this.meta.GeographicOption.featureTitles;
            this.searchField = this.meta.GeographicOption.searchField;
            this.formValueField = this.meta.GeographicOption.formValueField;
            this.successMsg = this.meta.GeographicOption.successMsg;
            this.errorMsg = this.meta.GeographicOption.errorMsg;
            this.searchFieldPlaceHolder = this.meta.GeographicOption.searchFieldPlaceHolder;

        }

        if (!this.meta.hidden && this.do_render)
            this.initMap();
    },
    data() {
        return {
            formInline: {
                search_value: ''
            },
            ruleInline: {
                search_value: [
                    {required: true, message: 'Хайх утга аа оруулна уу', trigger: 'blur'}
                ]
            },
            current: null,
            new_point: {
                lng: null,
                lat: null
            },
            attributes: [],
            geometryType: "point",
            points: [],
            openSide: false,
            allowMultiGeometryTypes: false,
            allowMultiGeometries: false,
            checkByArea: false,
            featureLayerUrl: '',
            featureTitles: '',
            searchField: '',
            successMsg: '',
            formValueField: '',
            errorMsg: '',
            searchLoading: false,
            center:
                {
                    lat: 47.91876971846709,
                    lng: 106.91736415028574
                },
            zoom: 15,
            map:
                null,
            currentBaseMap:
                0,
            baseMaps:
                [
                    {
                        title: 'iMap',
                        thumb: '/webgis/images/baseMaps/googleSatellite.jpg',
                        baseMap: L.tileLayer('https://cloudgis.mn/map01/Tilemap/demo/{z}/{x}/{y}.png?mskey=12345', {
                            name: 'imap',
                            maxZoom: 20,
                            subdomains: ['mt0', 'mt1', 'mt2', 'mt3']
                        }),
                    },
                    {
                        title: 'Google Сансрын',
                        thumb: '/webgis/images/baseMaps/googleSatellite.jpg',
                        baseMap: L.tileLayer('https://{s}.google.com/vt/lyrs=s,h&x={x}&y={y}&z={z}', {
                            name: 'googleil',
                            maxZoom: 20,
                            subdomains: ['mt0', 'mt1', 'mt2', 'mt3']
                        }),
                    },
                    {
                        title: 'Google Гудамж',
                        thumb: '/webgis/images/baseMaps/googleStreets.jpg',
                        baseMap: L.tileLayer('https://{s}.google.com/vt/lyrs=m&x={x}&y={y}&z={z}', {
                            name: 'googlei',
                            maxZoom: 20,
                            subdomains: ['mt0', 'mt1', 'mt2', 'mt3']
                        }),
                    },
                    {
                        title: 'Open Street Map',
                        baseMap: L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
                            name: 'osm',
                            maxZoom: 19,
                            subdomains: ['a', 'b', 'c']
                        }),
                        thumb: '/webgis/images/baseMaps/openstreet.jpg',
                    }
                ],
            draw: null,
            layer: new L.FeatureGroup(),
            searchLayer: null,
            drawValue: null
        };
    },
    beforeMount() {
        if (this.meta.options) {
            this.meta.options = null;
        }
    },
    methods: {
        removeEditeLauyer() {
            this.layer.eachLayer((layer) => {

                this.layer.removeLayer(layer);
            });


            this.cancelGraphic()
        },
        clearSearchLayer() {
            if (this.searchLayer) {
                if (this.map.hasLayer(this.searchLayer)) {
                    this.map.removeLayer(this.searchLayer);
                }


                this.searchLayer = null;
            }

        },
        handleSubmit() {

            let search_value = this.model.form[this.formValueField];
            if (search_value) {
                if(this.do_render){
                    this.searchLoading = true;
                    var instance = axios.create({
                        headers: {
                            common: {},
                        },
                    });

                    instance.get(`/api/get-region/${search_value}`).then(res => {
                        if (res.data.object_id >= 1) {
                            let url = `${this.featureLayerUrl}/query?returnGeometry=true&where=${this.searchField}=${res.data.object_id}&outFields=*&f=json&outSR=4326`;


                            instance.get(url, {
                                transformRequest: (data, headers) => {

                                    delete headers.common['X-CSRF-TOKEN']
                                    delete headers['X-CSRF-TOKEN']
                                    delete headers['Authorization']
                                    delete headers.common['X-Requested-With']
                                    delete headers.common['Authorization']
                                }
                            }).then(res => {


                                if (res.data.features.length >= 1) {
                                    var latlngs = res.data.features[0].geometry.rings[0];
                                    let rings = [];


                                    res.data.features[0].geometry.rings.forEach(ring => {

                                        let newLl = [];
                                        ring.forEach(ll => {
                                            newLl.push([ll[1], ll[0]])
                                        });

                                        rings.push(newLl);
                                    })


                                    this.clearSearchLayer();

                                    this.searchLayer = L.polygon(rings, {color: 'green'}).addTo(this.map);

                                    this.searchLayer.bindPopup(L.Util.template(this.featureTitles, res.data.features[0].attributes));

                                    let bounds = this.searchLayer.getBounds();
                                    this.map.fitBounds(bounds);

                                    this.searchLoading = false;
                                } else {
                                    this.searchLoading = false;
                                    this.clearSearchLayer();

                                    notification["error"]({
                                        message: this.lang.noSiteFound,
                                    });
                                }


                            })
                        }
                    });
                }


            }
        },
        cancelGraphic() {
            this.points = [];
            this.current = null;
            this.map.closePopup();
            this.openSide = false;
            let data = this.getData();
// this.clearSearchLayer();
            if (this.checkByArea && data) {
                if (this.searchLayer) {

                    let editAbleLayer = this.layer.toGeoJSON();
                    let searchFeature = this.searchLayer.toGeoJSON();

                    let not_inside = true;
                    searchFeature.geometry.coordinates.forEach(polygon_ => {
                        let searchLayer = polygon([polygon_]);
                        editAbleLayer.features.forEach((layer) => {

                            if (this.geometryType == 'polygon') {
                                layer.geometry.coordinates[0].map(lat_lng => {
                                    let pointNew = point(lat_lng);
                                    if (inside(pointNew, searchLayer)) {
                                        not_inside = false;
                                    }
                                })
                            }
                            if (this.geometryType == 'point') {
                                let pointNew = point(layer.geometry.coordinates);
                                if (inside(pointNew, searchLayer)) {
                                    not_inside = false;
                                }

                            }

                        });
                    });

                    if (!not_inside) {

                        this.model.form[this.model.component] = data;
                        notification["success"]({
                            message: this.successMsg,

                        });

                    } else {

                        this.model.form[this.model.component] = null;
                        notification["error"]({
                            message: this.errorMsg,

                        });

                    }


                } else {
                    this.layer.eachLayer((layer) => {

                        this.map.removeLayer(layer);

                    });
                    this.model.form[this.model.component] = null;
                    notification["error"]({
                        message: this.lang.theSiteHasNotBeenSelected,

                    });

                }
            } else
                this.model.form[this.model.component] = data;
        },
        createFeature(layer) {
            let geoJson = layer.toGeoJSON();
            let get_data = L.geoJSON(geoJson);
            get_data.eachLayer((l) => {
                this.layer.addLayer(this.setLayerOptions(l));
            });
        },
        setLayerOptions(layer) {

            layer.on('click', (e) => this.geometrySelectEvent(e));


            let properties = {}
            let popupForm = L.DomUtil.create('div');


            this.attributes.map(attribute => {

                properties[attribute] = layer.feature.properties[attribute] ? layer.feature.properties[attribute] : '';


                let foromEl = L.DomUtil.create('div');
                let line = L.DomUtil.create('br');
                let label = L.DomUtil.create('label');

                label.innerText = attribute;

                let input = L.DomUtil.create('input',);

                input.value = properties[attribute];
                L.DomEvent.addListener(input, 'change', (e) => {

                    properties[attribute] = input.value;
                });

                foromEl.appendChild(label);
                foromEl.appendChild(line);
                foromEl.appendChild(input);
                popupForm.appendChild(foromEl);
            });

            if (!this.checkByArea)
                layer.bindPopup(popupForm);

            layer.feature.properties = properties;

            return layer;
        },
        saveGraphic() {
            if (this.current) {
                if (this.geometryType == 'point') {
                    if (this.IsValidCoordinates(this.points[0])) {
                        let point = L.latLng([this.points[0][1], this.points[0][0]]);
                        this.current.setLatLng(point);
                    }
                } else {
                    let points = [];
                    this.points.map(point => {

                        if (this.IsValidCoordinates(point)) {
                            points.push(L.latLng([point[1], point[0]]))
                        }
                    });
                    this.current.setLatLngs(points);
                }
                this.cancelGraphic();
                return true;
            }

            let layer = null;
            if (this.geometryType == 'point') {
                if (this.IsValidCoordinates(this.points[0])) {
                    layer = L.marker([this.points[0][1], this.points[0][0]]);
                }


            } else {
                let points = []
                this.points.map(point => {
                    if (this.IsValidCoordinates(point)) {
                        points.push([point[1], point[0]])
                    }
                });


                if (this.geometryType == 'line') {

                    layer = new L.polyline(points);

                }
                if (this.geometryType == 'polygon') {

                    layer = new L.polygon(points);


                }
            }

            if (layer) {

                this.createFeature(layer)

            }


            this.cancelGraphic();
        },
        destroy() {

            this.meta.options = null;

            if (this.map !== null) {


                this.layer.eachLayer((layer) => {

                    this.map.removeLayer(layer);
                });
                this.map.removeLayer(this.layer);


                this.layer = new L.FeatureGroup();
                this.map.remove();

                this.map = null;
                var node = document.getElementById(`geographic_${this.model.component}`);

                var parent = node.parentNode;

                parent.removeChild(node);

                var div = document.createElement("div");
                div.setAttribute("id", `geographic_${this.model.component}`);
                div.setAttribute("class", `geographic-mapx`);

                parent.appendChild(div);


            }
        },
        initMap() {

            L.Marker.prototype.options.icon = iconDefault;


            L.Draw.Marker.prototype.options.icon = iconDefault;


            var container = L.DomUtil.get(`geographic_${this.model.component}`);
            if(container != null){
                container._leaflet_id = null;
            }
            this.map = L.map(`geographic_${this.model.component}`).setView([this.center.lat, this.center.lng], this.zoom);

            this.map.addLayer(this.baseMaps[this.currentBaseMap].baseMap);

            if (!this.itemValue) {

                this.layer.eachLayer((layer) => {

                    this.map.removeLayer(layer);
                });
                this.map.removeLayer(this.layer);
            }


            this.draw = new L.Control.Draw({
                draw: {
                    polygon: !this.allowMultiGeometryTypes ? this.geometryType === 'polygon' : true,
                    rectangle: !this.allowMultiGeometryTypes ? this.geometryType === 'polygon' : true,
                    marker: !this.allowMultiGeometryTypes ? this.geometryType === 'point' : true,
                    polyline: !this.allowMultiGeometryTypes ? this.geometryType === 'line' : true,
                    circle: false,
                    circlemarker: false,
                },
                edit: {
                    featureGroup: this.layer, //REQUIRED!!
                    remove: !(this.meta && this.meta.disabled)
                },


            });




            this.map.addControl(this.draw);

            L.control.scale().addTo(this.map);



            this.setElement();

            this.map.on(L.Draw.Event.CREATED, (e) => {
                let type = e.layerType,
                    layer = e.layer;

                if(this.meta && this.meta.disabled){
                    alert(this.lang.thereZNoRightChangeInformation);
                } else {
                    this.createFeature(layer);
                    this.cancelGraphic();
                }

            });

            this.map.on(L.Draw.Event.DELETED, (e) => {
                if(this.meta && this.meta.disabled){
                    alert(this.lang.thereZNoRightChangeInformation);
                } else {
                    this.cancelGraphic();
                }
            });
            this.map.on(L.Draw.Event.EDITED, (e) => {
                if(this.meta && this.meta.disabled){
                    alert(this.lang.thereZNoRightChangeInformation);
                } else {
                    this.cancelGraphic();
                }
            });



            if (this.formValueField && this.checkByArea) {

                if (this.model.form[this.formValueField]) {
                    if(this.do_render){
                        this.handleSubmit();
                    }

                }
            }
        },
        setElement(){
            if (this.itemValue) {

                try {
                    let geoJson = JSON.parse(this.itemValue);


                    if (this.geometryType === 'point') {
                        geoJson = {
                            type: "FeatureCollection",
                            features: [
                                {
                                    type: "Feature",
                                    properties: {},
                                    geometry: {
                                        coordinates: [geoJson.lng, geoJson.lat],
                                        type: "Point"
                                    }
                                }
                            ]
                        }


                    }
                    let init_data = L.geoJSON(geoJson);
                    init_data.eachLayer((l) => {

                        this.layer.addLayer(this.setLayerOptions(l));
                    });
                    this.map.fitBounds(this.layer.getBounds());
                } catch(e) {
                    console.log(e)
                }



            }
            this.map.addLayer(this.layer);
        },
        geometrySelectEvent(e) {
            if(this.meta && this.meta.disabled){

            } else {
                this.current = e.target;
                this.points = [];
                if (e.target._latlngs) {


                    let points = [];
                    if (e.target._latlngs[0] instanceof Array) {

                        this.geometryType = 'polygon';
                        points = e.target.getLatLngs()[0];

                    } else {
                        this.geometryType = 'line';
                        points = e.target.getLatLngs();
                    }

                    points.map(point => {

                        this.points.push([
                            point.lng, point.lat
                        ]);
                    });

                } else {
                    this.geometryType = 'point';
                    let point = e.target.getLatLng();

                    this.points.push([
                        point.lng, point.lat
                    ]);

                }
                this.openSide = true;
            }

        },
        getData() {
            let geoJson = this.layer.toGeoJSON();

            if (geoJson.features.length >= 2) {
                let layerIndex = 0;
                this.layer.eachLayer((layer) => {
                    layerIndex = layerIndex + 1
                    if (layerIndex < geoJson.features.length) {
                        this.layer.removeLayer(layer);
                    }

                });
            }
            geoJson = this.layer.toGeoJSON();

            if (geoJson.features.length >= 1) {

                if (this.geometryType == 'point') {
                    this.drawValue = geoJson
                    let point = {
                        lat: geoJson.features[0].geometry.coordinates[1],
                        lng: geoJson.features[0].geometry.coordinates[0]
                    }
                    return JSON.stringify(point);
                } else {
                    this.drawValue = geoJson
                    return JSON.stringify(geoJson);
                }

            } else {
                return null;
            }


        },
        changeBaseMap(baseMapIndex) {

            this.map.removeLayer(this.baseMaps[this.currentBaseMap].baseMap);

            this.currentBaseMap = baseMapIndex;


            this.map.options.maxZoom = 20;

            this.map.addLayer(this.baseMaps[baseMapIndex].baseMap);

        },
        createNewGeometry() {
            let source_value = null;
            if (this.new_point.lng !== null && this.new_point.lat === null) {
                source_value = this.new_point.lng;
            }
            if (this.new_point.lng === null && this.new_point.lat !== null) {
                source_value = this.new_point.lat;
            }
            if (this.new_point.lng !== null && this.new_point.lat !== null) {
                this.new_point.lng = this.new_point.lng * 1;
                this.new_point.lat = this.new_point.lat * 1;
                if (this.IsValidCoordinates([this.new_point.lng, this.new_point.lat])) {

                    this.points.push([this.new_point.lng, this.new_point.lat]);

                    this.resetInput()
                }
            }


            if (source_value) {

                let coordinates = source_value.split(' ');

                if (coordinates.length >= 1) {
                    let is_not_paste = false;
                    coordinates.map(coordinate => {
                        let lat_lng = coordinate.split('\t');
                        if (lat_lng.length >= 2) {
                            lat_lng[0] = lat_lng[0] * 1;
                            lat_lng[1] = lat_lng[1] * 1;
                            if (this.IsValidCoordinates([lat_lng[0], lat_lng[1]])) {

                                this.points.push(lat_lng);
                            }
                        } else {
                            is_not_paste = true
                        }

                    });
                    if (!is_not_paste) {
                        this.resetInput()
                    }

                }


            }

        },
        resetInput() {
            this.new_point = {
                lng: null,
                lat: null
            };
            this.$refs.lng.focus();

        },
        IsValidCoordinates(lng_lat) {
            if (lng_lat[0] && lng_lat[1]) {
                lng_lat[0] = lng_lat[0] * 1;
                lng_lat[1] = lng_lat[1] * 1;
                return this.longitude(lng_lat[0]) && this.latitude(lng_lat[1])
            } else return false

        },
        longitude(lon) {
            return !!(this.isNumber(lon) && this.between(lon, -180, 180))
        },
        latitude(lat) {
            return !!(this.isNumber(lat) && this.between(lat, -90, 90))
        },
        isNumber(n) {
            return typeof n === 'number'
        },
        between(value, n1, n2) {
            return n1 <= value && n2 >= value
        }
    },

    watch: {
        drawValue(value, oldValue) {
            //console.log(value, oldValue)
        },
        value_field_form(value, oldValue) {
            // console.log(value, oldValue, "this is form value")
            if (value) {
                this.handleSubmit();
            }

        },
        itemValue(value, oldValue) {


            if(value && !oldValue){
                this.setElement();
            }
            if((!value && !oldValue && this.editMode)){
               // this.setElement();
                this.initMap();
            }

        },
        do_render(value) {

            if (value) {
                if (!this.editMode) {
                    this.initMap();
                } else{
                    if(this.itemValue === null){
                        this.initMap();
                    }
                }


            } else {

                this.destroy();


            }
        },
        options() {
            if (this.meta.options) {
                this.formInline.search_value = this.meta.options;
                this.handleSubmit();
            }
        }
    }

};
</script>
<style lang="scss">
@import "leaflet/dist/leaflet.css";


</style>
