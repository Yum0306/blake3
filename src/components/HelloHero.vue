<template>
    <div class="container">
        <div class="card">
            <div class="card-body">
                <div class="row">
                    <div class="col-md-12">
                        <div class="form-group">
                            <label for="document">文件上传</label>
                            <input type="file"  @click="clear" class="form-control" accept='*' placeholder="Select file" @change="handleFiles"/>
                        </div>
                        <div class="form-group">
                            <label for="chunkSize">分片大小</label>
                            <input type="text"  class="form-control" value="1048576"/>
                        </div>
                        <!-- <div class="form-group row">
                            <div class="col-4">
                                Time shifting ordering
                            </div>
                            <div class="col-2">
                                <label class="switch">
                                    <input type="checkbox" id="switchMode" checked>
                                    <span class="slider round"></span>
                                </label>
                            </div>
                            <div class="col-4">
                                Memory buffered ordering
                            </div>
                        </div> -->
                        <div class="form-group">
                            <label for="hash">哈希值</label>
                            <input type="text" v-model="hashCode" class="form-control"/>
                        </div>
                        <div class="form-group">
                            <label for="fileSize">文件大小</label>
                            <input type="text"  v-model="fileSize" class="form-control"/>
                        </div>
                        <div class="form-group">
                            <label for="timeStart">开始时间</label>
                            <input type="text"  v-model="timeStart" class="form-control"/>
                        </div>
                        <div class="form-group">
                            <label for="timeEnd">结束时间</label>
                            <input type="text"  v-model="timeEnd" class="form-control"/>
                        </div>
                        <div class="form-group">
                            <label for="timeDelta">耗时</label>
                            <input type="text"  v-model="timeDelta" class="form-control"/>
                        </div>

                        <div class="form-group">
                            <label for="chunkTotal">总片数</label>
                            <input type="text" id="chunkTotal" v-model="chunkTotal" class="form-control"/>
                        </div>
                        <!-- <div class="form-group">
                            <label for="chunkReorder">Chunks reordered</label>
                            <input type="text" id="chunkReorder" v-model="chunkReorder" class="form-control"/>
                        </div> -->
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>
  
<script>
// import $ from 'jquery';
import CryptoJS from 'crypto-js';

Array.prototype.remove = Array.prototype.remove || function(val){
    var i = this.length;
    while(i--){
        if (this[i] === val){
            this.splice(i,1);
        }
    }
};
export default {
    data() {
        return {
            chunkSize: 1024 * 1024 * 2, // bytes
            timeout: 10, // millisec
            lastOffset: 0,
            chunkReorder: 0,
            chunkTotal: 0,
            previous: [],
            timestr1: '',
            hashCode: '',
            fileSize: '',
            timeStart:'',
            timeEnd: '',
            timeDelta: '',
        }
    },
    mounted() {
        // var inputElement = document.getElementById("document");
        // inputElement.addEventListener("change", this.handleFiles, false);

        // inputElement.click(function(){
            // this.clear();
        // })

    },
    methods: {
        handleFiles(ev) {
            var file = ev.target.files[0];
            if(file===undefined){
                return;
            }
            var SHA256 = CryptoJS.algo.SHA256.create();
            var counter = 0;
            var self = this;

            var timeStart = new Date().getTime();
            var timeEnd = 0;
            this.timeStart = new Date(timeStart)
            let humanFileSize = self.humanFileSize(file.size,true);
            self.fileSize = file.size
            console.log("humanFileSize:",humanFileSize)

            self.loading(file,
                function (data) {
                    var wordBuffer = CryptoJS.lib.WordArray.create(data);
                    SHA256.update(wordBuffer);
                    counter += data.byteLength;
                    console.log((( counter / file.size)*100).toFixed(0) + '%');
                }, function () {
                    var encrypted = SHA256.finalize().toString();
                    console.log("encrypted:",encrypted)
                    timeEnd = new Date().getTime();
                    self.hashCode = encrypted
                    self.timeStart = new Date(timeStart);
                    self.timeEnd = new Date(timeEnd);
                    self.timeDelta = ((timeEnd-timeStart)/1000+' sec');
                });

        },
        clear(){
            this.timeStart = '';
            this.lastOffset = 0;
            this.chunkReorder = 0;
            this.chunkTotal = 0;
        },
        loading(file, callbackProgress, callbackFinal) {
            let self = this;
            var offset = 0;
            var size = this.chunkSize;
            var partial;
            var index = 0;

            console.log("file.size:",file.size)
            if(file.size===0){
                callbackFinal();
            }
            while (offset < file.size) {
                partial = file.slice(offset, offset+size);
                var reader = new FileReader;
                reader.size = self.chunkSize;
                reader.offset = offset;
                reader.index = index;
                reader.onload = function(evt) {
                    console.log("onload")
                    self.callbackRead(this, file, evt, callbackProgress, callbackFinal);
                };
                reader.readAsArrayBuffer(partial);
                offset += self.chunkSize;
                index += 1;
            }
        },
        callbackRead(obj, file, evt, callbackProgress, callbackFinal){
            // if( $("#switchMode").is(':checked') ){
            //     this.callbackRead_buffered(obj, file, evt, callbackProgress, callbackFinal);
            // } else {
                this.callbackRead_waiting(obj, file, evt, callbackProgress, callbackFinal);
            // }
        },
        // time reordering
        callbackRead_waiting(reader, file, evt, callbackProgress, callbackFinal){
            let self = this;
            if(this.lastOffset === reader.offset) {
                console.log("[",reader.size,"]",reader.offset,'->', reader.offset+reader.size,"");
                this.lastOffset = reader.offset+reader.size;
                callbackProgress(evt.target.result);
                if ( reader.offset + reader.size >= file.size ){
                    this.lastOffset = 0;
                    callbackFinal();
                }
                this.chunkTotal++;
            } else {
                console.log("[",reader.size,"]",reader.offset,'->', reader.offset+reader.size,"wait");
                setTimeout(function () {
                    self.callbackRead_waiting(reader,file,evt, callbackProgress, callbackFinal);
                }, this.timeout);
                this.chunkReorder++;
            }
        },
        // memory reordering
        callbackRead_buffered(reader, file, evt, callbackProgress, callbackFinal){
            let self = this;
            this.chunkTotal++;

            if(this.lastOffset !== reader.offset){
                // out of order
                console.log("[",reader.size,"]",reader.offset,'->', reader.offset+reader.size,">>buffer");
                self.previous.push({ offset: reader.offset, size: reader.size, result: reader.result});
                this.chunkReorder++;
                return;
            }

            function parseResult(offset, size, result) {
                self.lastOffset = offset + size;
                callbackProgress(result);
                if (offset + size >= file.size) {
                    self.lastOffset = 0;
                    callbackFinal();
                }
            }

            // in order
            console.log("[",reader.size,"]",reader.offset,'->', reader.offset+reader.size,"");
            parseResult(reader.offset, reader.size, reader.result);

            // resolve previous buffered
            var buffered = [{}]
            while (buffered.length > 0) {
                buffered = self.previous.filter(function (item) {
                    return item.offset === self.lastOffset;
                });
                buffered.forEach(function (item) {
                    console.log("[", item.size, "]", item.offset, '->', item.offset + item.size, "<<buffer");
                    parseResult(item.offset, item.size, item.result);
                    self.previous.remove(item);
                })
            }

        },
        // Human file size
        humanFileSize(bytes, si) {
            var thresh = si ? 1000 : 1024;
            if (Math.abs(bytes) < thresh) {
                return bytes + ' B';
            }
            var units = si
                ? ['kB', 'MB', 'GB', 'TB', 'PB', 'EB', 'ZB', 'YB']
                : ['KiB', 'MiB', 'GiB', 'TiB', 'PiB', 'EiB', 'ZiB', 'YiB'];
            var u = -1;
            do {
                bytes /= thresh;
                ++u;
            } while (Math.abs(bytes) >= thresh && u < units.length - 1);
            return bytes.toFixed(1) + ' ' + units[u];
        }
    }
    

}
</script>

<style scoped>
    /* The switch - the box around the slider */
    .switch {
        position: relative;
        display: inline-block;
        width: 60px;
        height: 34px;
    }

    /* Hide default HTML checkbox */
    .switch input {display:none;}

    /* The slider */
    .slider {
        position: absolute;
        cursor: pointer;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        background-color: #ccc;
        -webkit-transition: .4s;
        transition: .4s;
    }

    .slider:before {
        position: absolute;
        content: "";
        height: 26px;
        width: 26px;
        left: 4px;
        bottom: 4px;
        background-color: white;
        -webkit-transition: .4s;
        transition: .4s;
    }

    input:checked + .slider {
        background-color: #2196F3;
    }

    input:focus + .slider {
        box-shadow: 0 0 1px #2196F3;
    }

    input:checked + .slider:before {
        -webkit-transform: translateX(26px);
        -ms-transform: translateX(26px);
        transform: translateX(26px);
    }

    /* Rounded sliders */
    .slider.round {
        border-radius: 34px;
    }

    .slider.round:before {
        border-radius: 50%;
    }
</style>