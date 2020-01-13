<template>
    <Page class="page">
        <ActionBar class="action-bar">
            <Label class="action-bar-title" text="Picture Gallery"></Label>
        </ActionBar>
        <StackLayout>
            <GridLayout columns="*,*" rows="auto" verticalAlignment="center">
                <Label col="0" row="0" text.decode="&#xf030; " @tap="takePicture" class="take-picture-icon fa" />
                <Label col="1" row="0" text.decode="&#xf1c5; " @tap="chooseImage" class="take-picture-icon fa" />
            </GridLayout>
            <ScrollView class="picture-gallery" orientation="vertical" height="100%">
                <StackLayout orientation="vertical">
                    <GridLayout v-for="image in arrayPictures" :key="image.id" columns="*" rows="*">
                        <Image class="gallery-item" col="0" row="0" :src="image" stretch="aspectFill" @tap="tapPicture(image)" />
                        <StackLayout v-if="image.note.length>0" col="0" row="0" class="note-picture-wrapper" @tap="tapPicture(image)">
                            <Label textWrap="true" verticalAlignement="bottom" :text="image.note" class="note-picture-text"></Label>
                        </StackLayout>
                    </GridLayout>
                </StackLayout>
            </ScrollView>
        </StackLayout>
    </Page>
</template>

<script>
const applicationSettings = require("application-settings");
const fsModule = require("tns-core-modules/file-system");
import { path, knownFolders } from "tns-core-modules/file-system";
const imageSourceModule = require("tns-core-modules/image-source");
const imageAssetModule = require("tns-core-modules/image-asset/image-asset");
const cameraModule = require("nativescript-camera");
const imagepicker = require("nativescript-imagepicker");
import { isAndroid, isIOS } from "tns-core-modules/platform";
const utilsModule = require("tns-core-modules/utils/utils");
import ImageDetails from "@/components/ImageDetails";

export default {
    computed: {},
    data() {
        return {
            arrayPictures: []
        };
    },
    methods: {
        tapPicture(image) {
            let navContextObj = {
                image: image,
                arrayPictures: this.arrayPictures,
                storeData: this.storeData
            };
            this.$navigateTo(ImageDetails, {
                animated: true,
                transition: {
                    name: "slideLeft",
                    curve: "easeInOut",
                    duration: 100
                },
                props: { navObject: navContextObj }
            });
        },
        takePicture() {
            let that = this;
            cameraModule
                .takePicture({
                    width: 500, //these are in device independent pixels
                    height: 500, //only one will be respected depending on os/device if
                    keepAspectRatio: true, //    keepAspectRatio is enabled.
                    saveToGallery: false //Don't save a copy in local gallery, ignored by some Android devices
                })
                .then(imageAsset => {
                    imageAsset.options.autoScaleFactor = false;
                    imageAsset.options.keepAspectRatio = true;
                    imageSourceModule
                        .fromAsset(imageAsset)
                        .then(imageSource => {
                            var ratio = 1,
                                newheight = imageSource.height,
                                newwidth = imageSource.width
                            if (imageSource.width > 500 && imageSource.height > 500) {
                                if (imageSource.width > imageSource.height) {
                                    newwidth = 500
                                    newheight = Math.round(500 * (imageSource.height / imageSource.width))
                                } else {
                                    newheight = 500
                                    newwidth = Math.round(500 * (imageSource.width / imageSource.height))
                                }
                            }
                            if (imageSource.width > 500 && imageSource.height > 500) {
                                if (isAndroid) {
                                    try {
                                        var downsampleOptions = new android.graphics.BitmapFactory.Options();
                                        var bitmap = android.graphics.BitmapFactory.decodeFile(imageAsset.android, null);
                                        var newBitmap = android.graphics.Bitmap.createScaledBitmap(bitmap, newwidth, newheight, true);
                                        imageSource.setNativeSource(newBitmap);
                                        let filename = "image-" + new Date().getTime() + ".jpg";
                                        let folder = fsModule.knownFolders.documents();
                                        let path = fsModule.path.join(folder.path, filename);
                                        imageSource.saveToFile(path, "jpeg");
                                        imageSource.filename = filename;
                                        imageSource.note = "";
                                        that.arrayPictures.unshift(imageSource);
                                        that.storeData();
                                    } catch (err) {
                                        console.error(err);
                                    }
                                } else {
                                    let manager = PHImageManager.defaultManager();
                                    let options = new PHImageRequestOptions();
                                    options.resizeMode =
                                        PHImageRequestOptionsResizeMode.Exact;
                                    options.deliveryMode = PHImageRequestOptionsDeliveryModeHighQualityFormat;
                                    manager.requestImageForAssetTargetSizeContentModeOptionsResultHandler(
                                        imageAsset.ios, { width: newwidth, height: newheight },
                                        PHImageContentModeAspectFill,
                                        options,
                                        function(result, info) {
                                            let folder = fsModule.knownFolders.documents();
                                            let path = fsModule.path.join(folder.path, filename);
                                            let filename = "image-" + new Date().getTime() + ".jpg";
                                            let newasset = new imageAssetModule.ImageAsset(result);
                                            imageSourceModule
                                                .fromAsset(newasset)
                                                .then(newimageSource => {
                                                    newimageSource.saveToFile(path, "jpeg");
                                                    newimageSource.filename = filename;
                                                    newimageSource.note = "";
                                                    that.arrayPictures.unshift(newimageSource);
                                                    that.storeData();
                                                });
                                        }
                                    );
                                }
                            } else {
                                let filename = "image" + "-" + new Date().getTime() + ".jpg";
                                let folder = fsModule.knownFolders.documents();
                                let path = fsModule.path.join(folder.path, filename);
                                imageSource.saveToFile(path, "jpeg");
                                imageSource.filename = filename;
                                imageSource.note = "";
                                that.arrayPictures.unshift(imageSource);
                                that.storeData();
                            }
                        })
                })
        },
        storeData() {
            let localArr = [];
            this.arrayPictures.forEach(entry => {
                localArr.unshift({ note: entry.note, filename: entry.filename });
            })
            applicationSettings.setString("localdata", JSON.stringify(localArr));
        },
        loadData() {
            let strData = applicationSettings.getString("localdata");
            if (strData && strData.length) {
                let localArr = JSON.parse(strData);
                localArr.forEach(entry => {
                    const folder = fsModule.knownFolders.documents();
                    const path = fsModule.path.join(folder.path, entry.filename);
                    var loadedImage = imageSourceModule.fromFile(path);
                    loadedImage.filename = entry.filename;
                    loadedImage.note = entry.note;
                    this.arrayPictures.unshift(loadedImage);
                })
            }
        },
        chooseImage() {
            let that = this
            let context = imagepicker.create({ mode: "single" });
            context
                .authorize()
                .then(() => {
                    return context.present();
                })
                .then(selection => {
                    const imageAsset = selection.length > 0 ? selection[0] : null;
                    imageAsset.options.autoScaleFactor = false;
                    imageAsset.options.keepAspectRatio = true;
                    imageSourceModule
                        .fromAsset(imageAsset)
                        .then(imageSource => {
                            var newheight = imageSource.height,
                                newwidth = imageSource.width
                            if (imageSource.width > 500 && imageSource.height > 500) {
                                if (imageSource.width > imageSource.height) {
                                    newwidth = 500
                                    newheight = Math.round(500 * (imageSource.height / imageSource.width))
                                } else {
                                    newheight = 500
                                    newwidth = Math.round(500 * (imageSource.width / imageSource.height))
                                }
                            }
                            if (imageSource.width > 500 && imageSource.height > 500) {
                                if (isAndroid) {
                                    try {
                                        var downsampleOptions = new android.graphics.BitmapFactory.Options();
                                        var bitmap = android.graphics.BitmapFactory.decodeFile(imageAsset.android, null);
                                        var newBitmap = android.graphics.Bitmap.createScaledBitmap(bitmap, newwidth, newheight, true);
                                        imageSource.setNativeSource(newBitmap);
                                        let filename = "image-" + new Date().getTime() + ".jpg";
                                        let folder = fsModule.knownFolders.documents();
                                        let path = fsModule.path.join(folder.path, filename);
                                        imageSource.saveToFile(path, "jpeg");
                                        imageSource.filename = filename;
                                        imageSource.note = "";
                                        that.arrayPictures.unshift(imageSource);
                                        that.storeData();
                                    } catch (err) {
                                        console.error(err);
                                    }
                                } else {
                                    let manager = PHImageManager.defaultManager();
                                    let options = new PHImageRequestOptions();
                                    options.resizeMode = PHImageRequestOptionsResizeMode.Exact;
                                    options.deliveryMode = PHImageRequestOptionsDeliveryModeHighQualityFormat;
                                    manager.requestImageForAssetTargetSizeContentModeOptionsResultHandler(
                                        imageAsset.ios, { width: newwidth, height: newheight },
                                        PHImageContentModeAspectFill,
                                        options,
                                        function(result, info) {
                                            let filename = "image" + "-" + new Date().getTime() + ".jpg";
                                            let folder = fsModule.knownFolders.documents();
                                            let path = fsModule.path.join(folder.path, filename);
                                            let newasset = new imageAssetModule.ImageAsset(result);
                                            newasset.options.autoScaleFactor = false;
                                            newasset.options.keepAspectRatio = true;
                                            newasset.height = newheight
                                            newasset.width = newwidth
                                            imageSourceModule
                                                .fromAsset(newasset)
                                                .then(newimageSource => {
                                                    newimageSource.saveToFile(
                                                        path,
                                                        "jpeg"
                                                    );
                                                    newimageSource.filename = filename;
                                                    newimageSource.note = "";
                                                    that.arrayPictures.unshift(newimageSource);
                                                    that.storeData();
                                                });
                                        }
                                    );
                                }
                            } else {
                                let folder = fsModule.knownFolders.documents();
                                let filename = "image-" + new Date().getTime() + ".jpg";
                                let path = fsModule.path.join(folder.path, filename);
                                imageSource.saveToFile(path, "jpeg");
                                imageSource.filename = filename;
                                imageSource.note = "";
                                that.arrayPictures.unshift(imageSource);
                                that.storeData();
                            }
                        })
                        .catch(err => {
                            console.error(err);
                        });
                });
        },
    },
    mounted() {
        cameraModule.requestPermissions().then( //request permissions for camera
            success => { //have permissions  
            },
            failure => { //no permissions for camera
            }
        )
        this.loadData()
    }
}
</script>

<style scoped lang="scss">
.take-picture-icon {
    horizontal-align: center;
    background-color: rgb(105, 105, 241);
    padding: 12;
    border-width: 1.2;
    border-color: black;
    border-radius: 14;
    margin-top: 20;
    margin-bottom: 20;
    color: white;
    font-size: 30;
    padding-left: 20;
}

.picture-gallery {
    margin-top: 20;
}

.gallery-item {
    margin: 10;
}

.note-picture-wrapper {
    background-color: #1a1919;
    opacity: 0.7;
    border-width: 1;
    border-radius: 8;
    color: #ffffff;
    border-color: #ffffff;
    margin: 15;
    vertical-align: bottom;
    horizontal-align: center;
}

.note-picture-text {
    font-size: 15;
    vertical-align: center;
    horizontal-align: center;
    padding: 4;
}
</style>