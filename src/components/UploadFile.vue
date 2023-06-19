<template>
  <div>
    <input type="file" @change="getFileContext" id="exampleFormControlFile1" />
    <p>{{ progress.toFixed(1) }}%</p>
  </div>
</template>

<script>
import axios from "axios";

const chunkSize = 1048576 * 100; // 3MB

export default {
  data() {
    return {
      showProgress: false,
      progress: 0,
      fileState: {
        fileSize: 0,
        fileId: "",
        totalChunks: 0,
        totalChunksUploaded: 0,
        startChunk: 0,
        endChunk: chunkSize,
        fileToUpload: null,
        uploadedBytes: 0,
      },
    };
  },
  methods: {
    getFileContext(event) {
      this.showProgress = true;
      this.progress = 0;
      this.resetState();
      const file = event.target.files[0];
      const fileId = `${file.size}-${file.lastModified}-${file.name}`;

      axios
        .get("http://localhost:3002/upload/status", {
          headers: {
            "x-file-name": fileId,
            "file-size": file.size,
          },
        })
        .then(({ data }) => {
          const uploadedBytes = data.uploaded;
          const bytesRemaining = file.size - uploadedBytes;
          const endingChunk = Math.min(uploadedBytes + chunkSize, file.size);
          this.fileState = {
            fileSize: file.size,
            fileId,
            totalChunks: Math.ceil(bytesRemaining / chunkSize),
            totalChunksUploaded: 0,
            startChunk: uploadedBytes,
            endChunk:
              endingChunk === this.fileState.fileSize
                ? endingChunk + 1
                : endingChunk,
            fileToUpload: file,
            uploadedBytes,
          };
        })
        .catch((error) => console.error("Status call failed ", error));
    },
    fileUpload(totalChunksUploaded) {
      const { totalChunks, fileToUpload, startChunk, endChunk, fileId } =
        this.fileState;
      if (totalChunksUploaded <= totalChunks) {
        const chunk = fileToUpload.slice(startChunk, endChunk);
        this.uploadChunk(chunk);
      } else {
        axios
          .post("http://localhost:3002/upload/complete", {
            headers: {
              "x-file-name": fileId,
            },
          })
          .then(this.resetState);
      }
    },
    uploadChunk(chunk) {
      const {
        fileId,
        startChunk,
        endChunk,
        fileSize,
        totalChunksUploaded,
        uploadedBytes,
      } = this.fileState;
      axios
        .post("http://localhost:3002/upload/files", chunk, {
          headers: {
            "x-file-name": fileId,
            "Content-Range": `bytes ${startChunk}-${endChunk}/${fileSize}`,
            "file-size": fileSize,
          },
        })
        .then(() => {
          const endingChunk = Math.min(endChunk + chunkSize, fileSize);
          this.fileState = {
            ...this.fileState,
            totalChunksUploaded: totalChunksUploaded + 1,
            startChunk: endChunk,
            endChunk: endingChunk === fileSize ? endingChunk + 1 : endingChunk,
            uploadedBytes: endingChunk,
          };
          const prog = fileSize ? (uploadedBytes / fileSize) * 100 : 0.1;
          this.progress = prog;
        });
    },
    resetState() {
      this.fileState = {
        fileSize: 0,
        fileId: "",
        totalChunks: 0,
        totalChunksUploaded: 0,
        startChunk: 0,
        endChunk: chunkSize,
        fileToUpload: null,
        uploadedBytes: 0,
      };
    },
  },
  watch: {
    fileState: {
      handler(newValue) {
        console.log("Uploaded:", newValue.uploadedBytes);
        if (newValue.fileSize > 0) {
          this.fileUpload(newValue.totalChunksUploaded);
        }
      },
      deep: true,
    },
  },
};
</script>
