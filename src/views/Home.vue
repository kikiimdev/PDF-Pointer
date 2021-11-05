<template>
    <v-main>
        <v-container>
            <v-card
                class="pa-4"
            >
                <v-layout column>
                    <v-file-input
                        v-model="selectedFile"
                        label="Upload PDF"
                        solo
                        flat
                        counter
                        accept=".pdf"
                        :show-size="1000"
                        background-color="grey lighten-3"
                        @change="onFileSelected"
                    />
                    <v-text-field
                        v-if="fileToUrl"
                        outlined
                        v-model="textModel"
                        label="Text To Add"
                        placeholder="I'am here!!!"
                    />
                    <v-btn
                        color="primary"
                        @click="addTextToPdf(selectedFile)"
                    >
                        Add Text To Pdf
                    </v-btn>
                    <div>
                        {{ dimension }}
                    </div>
                    <div>
                        {{ position }}
                    </div>

                    <v-layout
                        v-if="fileToUrl"
                        column
                        style="position: relative; width: 100%; height: 100%;"
                    >

                        <pdf
                            v-if="fileToUrl"
                            ref="pdfContainer"
                            :key="fileToUrl"
                            :src="fileToUrl"
                            :text="false"
                            :resize="true"
                            style="width: 100%; height: 100%"
                        >
                        </pdf>
                        <div
                            :style="`
                                width: 100%; 
                                height: 100%; 
                                position: absolute; 
                                background-color: rgba(255, 0 , 0, .0)
                            `"
                            @click="clickHandler"
                        />
                        <v-icon
                            v-if="fileToUrl"
                            :key="clickCounter"
                            color="error"
                            size="24"
                            v-text="'mdi-plus'"
                            :style="`
                                position: absolute;
                                left: ${position.containerX - 12}px;
                                top: ${position.containerY - 12}px;
                                pointer-events: none;
                            `"
                        />
                    </v-layout>
                </v-layout>
            </v-card>
        </v-container>
    </v-main>
</template>

<script lang="ts">
import { defineComponent, Ref, ref } from '@vue/composition-api'
// @ts-ignore
import pdf from "pdfvuer"
import { PDFDocument, rgb } from 'pdf-lib';

export default defineComponent({
    components: { pdf },
    name: 'Home',
    setup() {
        const pdfContainer = ref()
        const textModel = ref("I'am here!!!")
        const selectedFile = ref(null)
        const fileToUrl: Ref<any> = ref(null)
        const position = ref({
            pdfX: 0,
            pdfY: 0,
            containerX: 0,
            containerY: 0,
        })
        const dimension = ref({
            pdfWidth: 0,
            pdfHeight: 0,
            containerWidth: 0,
            containerHeight: 0,
            heightRatio: 1,
            widthRatio: 1,
        })
        const clickCounter = ref(0)

        function getPdfContainerDimension(el: any) {
            const height = el.$el.clientHeight
            const width = el.$el.clientWidth

            return { height, width }
        }

        async function addTextToPdf(file: any) {
            fileToUrl.value = URL.createObjectURL(file)
            let reader: any = new FileReader();
            reader.onload = async function(e: any) {
                let arrayBuffer = new Uint8Array(reader.result);
                await editPdf(arrayBuffer)
            }

            reader.readAsArrayBuffer(file);
        } 

        async function onFileSelected(file: any) {
            fileToUrl.value = URL.createObjectURL(file)
            let reader: any = new FileReader();
            reader.onload = async function(e: any) {
                let arrayBuffer = new Uint8Array(reader.result);

                const docDimension: any = await getPdfDimension(arrayBuffer).catch(err => console.log('Something wrong!', err.message))
                dimension.value.pdfWidth = docDimension.width
                dimension.value.pdfHeight = docDimension.height

                const containerDimension = getPdfContainerDimension(pdfContainer.value)
                dimension.value.containerWidth = containerDimension.width
                dimension.value.containerHeight = containerDimension.height

                dimension.value.heightRatio = containerDimension.height / docDimension.height
                dimension.value.widthRatio = containerDimension.width / docDimension.width
            }

            reader.readAsArrayBuffer(file);
            
        }

        function clickHandler(event: any) {
            // console.log(event);
        
            const { layerX, layerY } = event
            position.value.containerX = layerX
            position.value.containerY = layerY

            const { heightRatio, widthRatio } = dimension.value
            position.value.pdfX = layerX / widthRatio
            position.value.pdfY = layerY / heightRatio

            clickCounter.value ++
        }

        async function getPdfDimension(file: any) {
            const pdfDoc = await PDFDocument.load(file)
            const pages = pdfDoc.getPages()
            const firstPage = pages[0]
            const { width, height } = firstPage.getSize()

            return { width, height }
        }

        async function editPdf(file: any) {
            const pdfDoc = await PDFDocument.load(file)
            const pages = pdfDoc.getPages()
            const firstPage = pages[0]
            const { width, height } = firstPage.getSize()
            const { pdfX, pdfY } = position.value

            // console.log({
            //     width,
            //     height,
            //     pdfX,
            //     pdfY
            // });

            let fontSize: number = 12

            let offsetX: number = 2
            let offsetY: number = fontSize / 2

            firstPage.drawText(textModel.value, {
                x: pdfX - offsetX,
                y: height - pdfY - offsetY,
                size: fontSize,
                color: rgb(0.95, 0.1, 0.1),
            })

            const modifiedPdf = await pdfDoc.save()
            downloadBlob(modifiedPdf, 'newPdf.pdf')
        }

        const downloadURL = (data: any, fileName: string) => {
            const a = document.createElement('a')
            a.href = data
            a.download = fileName
            document.body.appendChild(a)
            a.style.display = 'none'
            a.click()
            a.remove()
        }

        const downloadBlob = (data: any, fileName: string, mimeType: string = 'application/pdf') => {

            const blob = new Blob([data], {
                type: mimeType
            })

            const url = window.URL.createObjectURL(blob)

            downloadURL(url, fileName)

            setTimeout(() => window.URL.revokeObjectURL(url), 1000)
        }

        return {
            textModel,
            pdfContainer,
            selectedFile,
            fileToUrl,
            onFileSelected,
            clickHandler,
            position,
            dimension,
            addTextToPdf,
            clickCounter
        }
    }
})
</script>

<style>

</style>
