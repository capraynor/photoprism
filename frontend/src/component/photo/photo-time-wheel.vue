<template>
    <div class="time-wheel">
        <div class="time-wheel__timeline" @mousemove="onMouseMove" @mouseleave="onMouseLeave">
            <div v-for="i in yearGroupedIDs" :key="`${i.Year}`" :style="{height: `${i.ids.length / ids.length * 100}%`}">
                {{ i.Year }}
            </div>
            <div class="time-wheel__hover-indicator" :style="{...hoverIndicator}"></div>
        <div class="time-wheel__scroll-indicator" :style="{...scrollIndicator}"></div>
        </div>


        <div v-if="showPreviewPanel" class="time-wheel__photoPreviewPanel" :style="{...previewPanelPosition}">
            <div v-for="index in (previewPhotoIndexes.endIndex - previewPhotoIndexes.startIndex)" v-if="!!photos[index + previewPhotoIndexes.startIndex]" class="time-wheel__photoPreviewPanel__photo"
            :style="`background-image: url(${photos[index + previewPhotoIndexes.startIndex].thumbnailUrl('tile_224')});`"
            >
                <div class="time-wheel__photoPreviewPanel__date">{{ `${photos[index + previewPhotoIndexes.startIndex].Year}-${photos[index + previewPhotoIndexes.startIndex].Month}-${photos[index + previewPhotoIndexes.startIndex].Day}` }}</div>
            </div>
        </div>
    </div>
</template>
<script>
export default {
    name: "PPhotoTimeWheel",
    props: {
        ensurePhotoLoaded: {
            type: Function,
            default: () => {}
        },
        photos: {
            type: Array,
            default: () => [],
        },
        ids: {
            type: Array,
            default: () => [],
        },
        onPhotoClicked: {
            type: Function,
            default: () => {}
        },
        startIndex: {
            type: Number,
            default: 0
        },
        endIndex: {
            type: Number,
            default: 0
        },
        
    },
    methods: {
        async onMouseMove(e){
            const rect = e.currentTarget.getBoundingClientRect();
            const mouseY = e.clientY - rect.top; 
            this.mousePosition.clientY =  e.clientY;
            const hoverIndicatorPosition = mouseY / e.currentTarget.offsetHeight;
            const hoverIndicatorTop = `${hoverIndicatorPosition * 100}%`;
            this.hoverIndicator.top = hoverIndicatorTop;
            this.hoverIndicator.display = "block"
            this.hoverIndicatorPosition = hoverIndicatorPosition;
            await this.ensurePhotoLoaded(this.previewPhotoIndexes.startIndex, this.previewPhotoIndexes.endIndex)
            this.showPreviewPanel = true;
        },

        onMouseLeave(){
            this.hoverIndicator.display = "none";
            this.showPreviewPanel = true;
        }

    },
    computed: {
        viewportPhotoCount(){
            return this.endIndex - this.startIndex + 1;
        },
        scrollIndicator() {
            return {
                height: `${this.viewportPhotoCount / this.ids.length * 100}%`,
                top:  `${this.startIndex / (this.ids.length - 1) * 100}%`
            }
        },
        previewPhotoIndexes() {
            var startIndex = Math.floor(this.hoverIndicatorPosition * (this.ids.length - 1));
            var endIndex = startIndex + 9;
            if(this.ids.length < endIndex){
                endIndex = this.ids.length;
            }
            return {
                startIndex,
                endIndex
            }
        },
        previewPanelPosition(){
            let clientY = this.mousePosition.clientY;
            let top = clientY;

            //Raynor @ May.14th, 2024: 450 in here copied from layout.css, .time-wheel__photoPreviewPanel
            if ((top + (450 / 2)) >  window.innerHeight){
                top = window.innerHeight - (450/2);
            }

            if ((top - (450 / 2)) <  0){
                top = 0 + (450/2);
            }

            

            return {
                top: `${top}px`
            }
        },
        yearGroupedIDs() {
            let result = [];
            this.ids.forEach((x, i) => {
                let groupedItem = result.find(r => r.Year === x.Year);
                if (!groupedItem){
                    groupedItem = {
                        Year: x.Year,
                        ids: []
                    }
                    result.push(groupedItem);
                }
                x.index = i;
                groupedItem.ids.push(x);
            });

            return result.sort(x => x.Year);
        },
        yearMonthGroupedIDs() {
            let result = []
            this.ids.forEach((x, i) => {
                let groupedItem = result.find(r => r.Month === x.Month && r.Year === x.Year);
                if (!groupedItem){
                    groupedItem = {
                        Month: x.Month,
                        PaddedMonth: String(x.Month).padStart(2, 0),
                        Year: x.Year,
                        ids: []
                    }
                    result.push(groupedItem);
                }
                x.index = i;
                groupedItem.ids.push(x);
            });


            const cmp = (a, b) => (a > b) - (a < b)
            result = result.sort((a, b) => {
                return cmp(a.Year, b.Year) || cmp(a.Month, b.Month);
            }).reverse();
            return result;
        }
    },
    data() {
        return {
            hoverIndicator: {
                top: "0%",
                display: "none",
            },
            hoverIndicatorPosition: 0,
            showPreviewPanel: false,
            mousePosition: {
                clientY: 0, 
                clientX: 0
            }
        }
    }
}
</script>