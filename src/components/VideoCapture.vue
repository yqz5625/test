<script setup lang="ts">
import { onMounted, onBeforeUnmount, ref, type Ref } from "vue";
import "../dynamsoft.config";
import { CameraEnhancer, CameraView, CaptureVisionRouter, MultiFrameResultCrossFilter } from "dynamsoft-barcode-reader-bundle";

const componentDestroyedErrorMsg = "VideoCapture Component Destroyed";

const cameraViewContainer: Ref<HTMLElement | null> = ref(null);

let resolveInit: () => void;
const pInit: Promise<void> = new Promise(r => { resolveInit = r });
let isDestroyed = false;

let cvRouter: CaptureVisionRouter;
let cameraEnhancer: CameraEnhancer;
let resultText = ref("");

onMounted(async () => {
  try {
    const setting = {
    "BarcodeFormatSpecificationOptions": [
        {
            "BarcodeFormatIds": [
                "BF_DATAMATRIX",
                "BF_QR_CODE",
                "BF_DEFAULT"
            ],
            "MirrorMode": "MM_BOTH",
            "BarcodeTextRegExPattern": "^[sS][A-Za-z0-9]+$",
            "Name": "bfs1-read-rate-first",
            "HasVerticalQuietZone": 0
        },
        {
            "BarcodeTextRegExPattern": "^[sS][A-Za-z0-9]+$",
            "Name": "bfs2-read-rate-first"
        }
    ],
    "BarcodeReaderTaskSettingOptions": [
        {
            "BarcodeFormatIds": [
                "BF_DATAMATRIX",
                "BF_QR_CODE",
                "BF_DEFAULT"
            ],
            "BarcodeFormatSpecificationNameArray": [
                "bfs1-read-rate-first",
                "bfs2-read-rate-first"
            ],
            "ExpectedBarcodesCount": 999,
            "MaxThreadsInOneTask": 4,
            "Name": "task-read-barcodes-read-rate",
            "SectionArray": [
                {
                    "ImageParameterName": "ip-read-barcodes-read-rate",
                    "Section": "ST_REGION_PREDETECTION",
                    "StageArray": [
                        {
                            "Stage": "SST_PREDETECT_REGIONS"
                        }
                    ]
                },
                {
                    "ImageParameterName": "ip-read-barcodes-read-rate",
                    "Section": "ST_BARCODE_LOCALIZATION",
                    "StageArray": [
                        {
                            "LocalizationModes": [
                                {
                                    "Mode": "LM_CONNECTED_BLOCKS"
                                },
                                {
                                    "Mode": "LM_LINES"
                                },
                                {
                                    "Mode": "LM_STATISTICS"
                                }
                            ],
                            "Stage": "SST_LOCALIZE_CANDIDATE_BARCODES"
                        },
                        {
                            "Stage": "SST_LOCALIZE_BARCODES"
                        }
                    ]
                },
                {
                    "ImageParameterName": "ip-read-barcodes-read-rate",
                    "Section": "ST_BARCODE_DECODING",
                    "StageArray": [
                        {
                            "Stage": "SST_RESIST_DEFORMATION"
                        },
                        {
                            "Stage": "SST_COMPLEMENT_BARCODE"
                        },
                        {
                            "Stage": "SST_SCALE_BARCODE_IMAGE"
                        },
                        {
                            "Stage": "SST_DECODE_BARCODES"
                        }
                    ]
                }
            ]
        }
    ],
    "CaptureVisionTemplates": [
        {
            "ImageROIProcessingNameArray": [
                "roi-read-barcodes-read-rate"
            ],
            "Name": "Test",
            "Timeout": 100000
        }
    ],
    "GlobalParameter": {
        "IntraOpNumThreads": 4
    },
    "ImageParameterOptions": [
        {
            "ApplicableStages": [
                {
                    "Stage": "SST_INPUT_COLOR_IMAGE"
                },
                {
                    "ImageScaleSetting": {
                        "EdgeLengthThreshold": 9999,
                        "ScaleType": "ST_SCALE_DOWN"
                    },
                    "Stage": "SST_SCALE_IMAGE"
                },
                {
                    "Stage": "SST_CONVERT_TO_GRAYSCALE"
                },
                {
                    "GrayscaleTransformationModes": [
                        {
                            "Mode": "GTM_ORIGINAL"
                        },
                        {
                            "Mode": "GTM_INVERTED"
                        }
                    ],
                    "Stage": "SST_TRANSFORM_GRAYSCALE"
                },
                {
                    "Stage": "SST_ENHANCE_GRAYSCALE"
                },
                {
                    "BinarizationModes": [
                        {
                            "BlockSizeX": 0,
                            "BlockSizeY": 0,
                            "Mode": "BM_LOCAL_BLOCK",
                            "ThresholdCompensation": 10
                        },
                        {
                            "Mode": "BM_LOCAL_BLOCK"
                        }
                    ],
                    "Stage": "SST_BINARIZE_IMAGE"
                },
                {
                    "Stage": "SST_DETECT_TEXTURE"
                },
                {
                    "Stage": "SST_REMOVE_TEXTURE_FROM_GRAYSCALE"
                },
                {
                    "Stage": "SST_BINARIZE_TEXTURE_REMOVED_GRAYSCALE"
                },
                {
                    "Stage": "SST_FIND_CONTOURS"
                },
                {
                    "Stage": "SST_DETECT_SHORTLINES"
                },
                {
                    "Stage": "SST_ASSEMBLE_LINES"
                },
                {
                    "Stage": "SST_DETECT_TEXT_ZONES"
                },
                {
                    "IfEraseTextZone": 1,
                    "Stage": "SST_REMOVE_TEXT_ZONES_FROM_BINARY"
                }
            ],
            "Name": "ip-read-barcodes-read-rate"
        }
    ],
    "TargetROIDefOptions": [
        {
            "Name": "roi-read-barcodes-read-rate",
            "TaskSettingNameArray": [
                "task-read-barcodes-read-rate"
            ]
        }
    ],
    "Version": "5.1"
}
    
    // Create a `CameraEnhancer` instance for camera control and a `CameraView` instance for UI control.
    const cameraView = await CameraView.createInstance();
    if (isDestroyed) { throw Error(componentDestroyedErrorMsg); } // Check if component is destroyed after every async

    cameraEnhancer = await CameraEnhancer.createInstance(cameraView);
    if (isDestroyed) { throw Error(componentDestroyedErrorMsg); }

    // Get default UI and append it to DOM.
    cameraViewContainer.value!.append(cameraView.getUIElement());

    // Create a `CaptureVisionRouter` instance and set `CameraEnhancer` instance as its image source.
    cvRouter = await CaptureVisionRouter.createInstance();
    cvRouter.initSettings(setting)
    if (isDestroyed) { throw Error(componentDestroyedErrorMsg); }
    cvRouter.setInput(cameraEnhancer);

    // Define a callback for results.
    cvRouter.addResultReceiver({
      onDecodedBarcodesReceived: (result) => {
        if (!result.barcodeResultItems.length) return;

        resultText.value = '';
        console.log(result);
        for (let item of result.barcodeResultItems) {
          resultText.value += `${item.formatString}: ${item.text}\n\n`;
        }
      }
    });

    // Filter out unchecked and duplicate results.
    const filter = new MultiFrameResultCrossFilter();
    // Filter out unchecked barcodes.
    filter.enableResultCrossVerification("barcode", true);
    // Filter out duplicate barcodes within 3 seconds.
    filter.enableResultDeduplication("barcode", true);
    await cvRouter.addResultFilter(filter);
    if (isDestroyed) { throw Error(componentDestroyedErrorMsg); }

    // Open camera and start scanning barcode.
    await cameraEnhancer.open();
    cameraView.setScanLaserVisible(true);
    if (isDestroyed) { throw Error(componentDestroyedErrorMsg); }
    await cvRouter.startCapturing("Test");
    if (isDestroyed) { throw Error(componentDestroyedErrorMsg); }

  } catch (ex: any) {

    if ((ex as Error)?.message === componentDestroyedErrorMsg) {
      console.log(componentDestroyedErrorMsg);
    } else {
      let errMsg = ex.message || ex;
      console.error(ex);
      alert(errMsg);
    }
  }

  // Resolve pInit promise once initialization is complete.
  resolveInit!();
});

// dispose cvRouter when it's no longer needed
onBeforeUnmount(async () => {
  isDestroyed = true;
  try {
    await pInit; // Wait for the pInit to complete before disposing resources.
    cvRouter?.dispose();
    cameraEnhancer?.dispose();
  } catch (_) { }
});
</script>

<template>
  <div>
    <div ref="cameraViewContainer" style="width: 100%; height: 70vh; background: #eee;"></div>
    <br />
    Results:
    <div class="results">{{ resultText }}</div>
  </div>
</template>

<style scoped>
.results {
  width: 100%;
  height: 10vh;
  overflow: auto;
  white-space: pre-wrap;
}
</style>
