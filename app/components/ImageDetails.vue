<template>
    <Page class="page" ref="page" actionBarHidden="false" backgroundSpanUnderStatusBar="true">
        <ActionBar class="action-bar" title="Picture Details">
            <NavigationButton text="Done" android.systemIcon="ic_menu_back" @tap="$navigateBack()" />
            <Label text.decode="&#xf019;" @tap="downloadImage()" class="take-picture-icon fa" />
        </ActionBar>
        <ScrollView orientation="vertical">
            <StackLayout>
                <Image class="picture-full" stretch="aspectFit" :src="curImage" />
                <GridLayout columns="*,*" rows="60,30,*,300">
                    <StackLayout col="0" row="0" class="delete-picture-icon-wrapper" @tap="editImage">
                        <Label verticalAlignement="bottom" text="edit" class="delete-picture-icon"></Label>
                    </StackLayout>
                    <StackLayout col="1" row="0" class="delete-picture-icon-wrapper" @tap="deletePicture">
                        <Label verticalAlignement="bottom" text="delete" class="delete-picture-icon"></Label>
                    </StackLayout>
                    <Label col="0" colSpan="2" row="1" text="Note:" class="section-label" />
                    <TextView col="0" colSpan="2" row="2" class="text-picture" hint="Add a note for this picture here" editable="true" v-model="navObject.image.note" />
                    <Label col="0" colSpan="2" row="3" text="" />
                </GridLayout>
            </StackLayout>
        </ScrollView>
    </Page>
</template>

<script>
const imageSourceModule = require("tns-core-modules/image-source");
const imageAssetModule = require("tns-core-modules/image-asset/image-asset");
const fsModule = require("tns-core-modules/file-system");
const applicationModule = require("application");
import { isAndroid, isIOS } from "tns-core-modules/platform";
import { PhotoEditor, PhotoEditorControl } from "nativescript-photo-editor";
const photoEditor = new PhotoEditor();
export default {
    name: "image-details-page",
    data() {
        return {
            curImage: '',
        };
    },
    props: {
        navObject: {
            type: Object,
        },
    },
    components: {},
    computed: {},
    created() {},
    beforeDestroy() {
        this.navObject.storeData()
    },
    mounted() {
        this.curImage = this.navObject.image
    },
    methods: {
        deletePicture() {
            let pictureIndex = this.navObject.arrayPictures.indexOf(this.navObject.image);
            this.navObject.arrayPictures.splice(pictureIndex, 1);
            this.$navigateBack()
        },
        editImage() {
            let that = this
            let folder = fsModule.knownFolders.documents();
            let path = fsModule.path.join(folder.path, this.navObject.image.filename);
            photoEditor
                .editPhoto({ imageSource: this.curImage })
                .then(newImage => {
                    newImage.filename = that.curImage.filename
                    newImage.note = that.curImage.note
                    that.curImage = newImage;
                    let pictureIndex = that.navObject.arrayPictures.indexOf(that.navObject.image);
                    that.navObject.arrayPictures.splice(pictureIndex, 1, newImage);
                    newImage.saveToFile(path, "jpeg");
                })
                .catch(e => {
                    console.error(e);
                });
        },
        downloadImage() {
            let that = this
            let folder = fsModule.knownFolders.documents();
            let path = fsModule.path.join(folder.path, this.navObject.image.filename);
            let imageSource = imageSourceModule.fromFile(path)
            if (isIOS) {
                PHPhotoLibrary.requestAuthorization((result) => {
                    if (result === PHAuthorizationStatus.Authorized) {
                        var CompletionTarget = NSObject.extend({
                            "thisImage:hasBeenSavedInPhotoAlbumWithError:usingContextInfo:": function(
                                image,
                                error,
                                context
                            ) {
                                if (error) {
                                    console.error("Unable to save to library, please try again.")
                                }
                            }
                        }, {
                            exposedMethods: {
                                "thisImage:hasBeenSavedInPhotoAlbumWithError:usingContextInfo:": {
                                    returns: interop.types.void,
                                    params: [UIImage, NSError, interop.Pointer]
                                }
                            }
                        });
                        var completionTarget = CompletionTarget.new();
                        UIImageWriteToSavedPhotosAlbum(
                            imageSource.ios,
                            completionTarget,
                            "thisImage:hasBeenSavedInPhotoAlbumWithError:usingContextInfo:",
                            null
                        );
                        console.log("Image saved to device")
                    } else {
                        if (isIOS) alert({ title: "Save Failed!", okButtonText: "OK", message: "Allow permission in Settings > Privacy > Photos to save an image to your device!" })
                        else alert("Save Failed! No permissions to save to this device.")
                    }
                });
            } else {
                function broadCast(imageFile) {
                    var mediaScanIntent = new android.content.Intent(
                        android.content.Intent.ACTION_MEDIA_SCANNER_SCAN_FILE
                    );
                    var contentUri = android.net.Uri.fromFile(imageFile);
                    mediaScanIntent.setData(contentUri);
                    applicationModule.android.foregroundActivity.sendBroadcast(
                        mediaScanIntent
                    );
                    alert("Image saved to device")
                }
                var folderPath = android.os.Environment.getExternalStoragePublicDirectory(
                    android.os.Environment.DIRECTORY_DOWNLOADS
                ).toString();
                let filename = "galleryimg_" + new Date().getTime() + ".jpg";
                let savepath = fsModule.path.join(folderPath, filename);
                var saved = imageSource.saveToFile(savepath, "jpeg");
                if (saved) {
                    broadCast(new java.io.File(savepath));
                } else {
                    alert("Error: Unable to save file!");
                }
            }
        },
    }
};
</script>

<style scoped>
.delete-picture-icon {
    font-size: 15;
    vertical-align: center;
    horizontal-align: center;
}

.delete-picture-icon-wrapper {
    background-color: #000000;
    border-width: 1;
    border-radius: 8;
    color: #ffffff;
    border-color: #ffffff;
    margin: 15;
    vertical-align: center;
    horizontal-align: right;
    height: 30;
    width: 60;
}

.text-picture {
    border-width: 1;
    border-style: solid;
    border-color: #01060c;
    height: 80;
    background-color: rgb(235, 233, 233);
}

.section-label {
    background-color: #292b2b;
    border-width: 1;
    border-style: solid;
    border-color: #01060c;
    color: white;
    padding-left: 10;
    padding-top: 5;
    padding-bottom: 5;
}

.picture-full {
    border-width: 1;
    border-color: gray;
}
</style>
