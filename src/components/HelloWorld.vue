<template>
  <div class="hello">
    <input type="file" @change="changeFile">
    
  </div>
</template>

<script>
// import xxhash from "xxhash-wasm";
// import * as blake3 from 'blake3/browser-async';
// const { hash, createHash } = require('blake3');
// import * as blake3 from 'blake3';
// import { blake3 } from '@noble/hashes/blake3';
// import { bytesToHex as toHex } from '@noble/hashes/utils';
export default {
  name: 'HelloWorld',
  props: {
    msg: String
  },
  methods: {
    changeFile(ev) {

      let s  = new Date().getTime()
      console.log("s:",s);
      let file = ev.target.files[0]
      console.log(file)
      // const reader = new FileReader();
      // reader.readAsArrayBuffer(ev.target.files[0]);
      // reader.onprogress = function(e){
      //    console.log("loaded:",e.loaded)
      // }
      // reader.onload = function (e) {
          // console.log('DONE', e.target.readyState); // readyState will be 2
          
          // console.log(e.target.result)
        const unit8array = new Uint8Array(ev.target.files[0])
        console.log(unit8array)

          // let test = await blake.load()
          

          // blake3.load().then(() => {
          //   console.log(blake3.keyedHash(unit8array).toString('hex'))
          // });
          
          console.log(toHex(blake3(unit8array)))
          
          // xxhash().then(hasher => {

          //         // 32-bit version
          //         console.log(hasher.h32(unit8array)); // 3998627172 (decimal representation)
          //         // For convenience, get hash as string of its zero-padded hex representation
          //         console.log(hasher.h32ToString(unit8array)); // "ee563564"

          //         // 64-bit version
          //         console.log(hasher.h64(unit8array)); // 5776724552493396044n (BigInt)
          //         // For convenience, get hash as string of its zero-padded hex representation
          //         console.log(hasher.h64ToString(unit8array)); // "502b0c5fc4a5704c"
          //         console.log("e:",(new Date().getTime()-s));
          //         ev.value = ''
          // });
          ev.value = ''
      // };
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
</style>
