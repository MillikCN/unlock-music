<template>
    <div id="app">
        <el-container>
            <el-main>
                <el-upload
                        :auto-upload="false"
                        :on-change="handleFile"
                        :show-file-list="false"
                        action=""
                        drag
                        multiple>
                    <i class="el-icon-upload"></i>
                    <div class="el-upload__text">将文件拖到此处，或<em>点击选择</em></div>
                    <div class="el-upload__tip" slot="tip">本工具仅在浏览器内对文件进行解锁，无需消耗流量</div>
                </el-upload>

                <el-row id="app-control">

                    <el-button @click="handleDownloadAll" icon="el-icon-download" plain>下载全部</el-button>
                    <el-button @click="handleDeleteAll" icon="el-icon-delete" plain type="danger">删除全部</el-button>

                </el-row>
                <audio :autoplay="playing_auto" :src="playing_url" controls></audio>


                <el-table :data="tableData" style="width: 100%">

                    <el-table-column label="封面">
                        <template slot-scope="scope">
                            <el-image :src="scope.row.picture" style="width: 100px; height: 100px">
                                <div class="image-slot el-image__error" slot="error">
                                    暂无封面
                                </div>
                            </el-image>
                        </template>
                    </el-table-column>
                    <el-table-column label="歌曲" sortable>
                        <template slot-scope="scope">
                            <span style="margin-left: 10px">{{ scope.row.title }}</span>
                        </template>
                    </el-table-column>
                    <el-table-column label="歌手" sortable>
                        <template slot-scope="scope">
                            <p>{{ scope.row.artist }}</p>
                        </template>
                    </el-table-column>
                    <el-table-column label="专辑" sortable>
                        <template slot-scope="scope">
                            <p>{{ scope.row.album }}</p>
                        </template>
                    </el-table-column>
                    <el-table-column label="操作">
                        <template slot-scope="scope">
                            <el-button @click="handlePlay(scope.$index, scope.row)"
                                       circle icon="el-icon-video-play" type="success">
                            </el-button>
                            <el-button @click="handleDownload(scope.row)"
                                       circle icon="el-icon-download">
                            </el-button>
                            <el-button @click="handleDelete(scope.$index, scope.row)"
                                       circle icon="el-icon-delete" type="danger">
                            </el-button>
                        </template>
                    </el-table-column>
                </el-table>
            </el-main>
            <el-footer id="app-footer">
                <el-row>
                    音乐解锁：移除已购音乐的加密保护。
                    目前支持网易云音乐(ncm)、QQ音乐(qmc, mflac, tkm)以及
                    <a href="https://github.com/ix64/unlock-music/blob/master/README.md" target="_blank">其他格式</a>。
                    <a href="https://github.com/ix64/unlock-music/wiki/使用提示" target="_blank">使用提示</a>
                </el-row>
                <el-row>
                    <span>Copyright &copy; 2019</span>
                    <a href="https://github.com/ix64" target="_blank">MengYX</a>
                    音乐解锁使用
                    <a href="https://github.com/ix64/unlock-music/blob/master/LICENSE" target="_blank">MIT许可协议</a>
                    开放
                    <a href="https://github.com/ix64/unlock-music" target="_blank">源代码</a>
                </el-row>
            </el-footer>
        </el-container>
    </div>
</template>

<script>

    const dec = require("./decrypt/common");
    export default {
        name: 'app',
        components: {},
        data() {
            return {
                activeIndex: '1',
                tableData: [],
                playing_url: "",
                playing_auto: false,
            }
        },
        mounted() {
            this.$nextTick(function () {
                this.finishLoad();
            });
        },
        methods: {
            finishLoad() {
                document.getElementById("loader-mask").remove();
                this.$notify.info({
                    title: '离线使用',
                    message: '我们使用PWA技术，无网络也能使用<br/>' +
                        '最近更新：支持bkcmp3/bkcflac/tkm<br/>' +
                        '点击查看 <a target="_blank" href="https://github.com/ix64/unlock-music/wiki/使用提示">使用提示</a>',
                    dangerouslyUseHTMLString: true,
                    duration: 10000,
                    position: 'top-left'
                });
            },
            handleFile(file) {

                (async () => {
                    let data = await dec.CommonDecrypt(file);
                    if (data.status) {
                        this.tableData.push(data);
                        this.$notify.success({
                            title: '解锁成功',
                            message: '成功解锁 ' + data.title,
                            duration: 3000
                        });
                        let _rp_data = [data.title, data.artist, data.album];
                        window._paq.push(["trackEvent", "Unlock", data.rawExt + "," + data.mime, JSON.stringify(_rp_data)]);
                    } else {
                        this.$notify.error({
                            title: '出现问题',
                            message: data.message + "，" + file.name +
                                '，参考<a target="_blank" href="https://github.com/ix64/unlock-music/wiki/使用提示">使用提示</a>',
                            dangerouslyUseHTMLString: true,
                            duration: 6000
                        });
                        window._paq.push(["trackEvent", "Error", data.message, file.name]);
                    }
                })();
            },
            handlePlay(index, row) {
                this.playing_url = row.file;
                this.playing_auto = true;
            },
            handleDelete(index, row) {
                URL.revokeObjectURL(row.file);
                URL.revokeObjectURL(row.picture);
                this.tableData.splice(index, 1);
            },
            handleDownload(row) {
                let a = document.createElement('a');
                a.href = row.file;
                a.download = row.filename;
                document.body.append(a);
                a.click();
                a.remove();
            },
            handleDeleteAll() {
                this.tableData.forEach(value => {
                    URL.revokeObjectURL(value.file);
                    URL.revokeObjectURL(value.picture);
                });
                this.tableData = [];
            },
            handleDownloadAll() {
                let index = 0;
                let c = setInterval(() => {
                    if (index < this.tableData.length) {
                        this.handleDownload(this.tableData[index]);
                        index++;
                    } else {
                        clearInterval(c);
                    }
                }, 1000);

            }
        }
    }

</script>

<style>
    #app {
        font-family: "Helvetica Neue", Helvetica, "PingFang SC",
        "Hiragino Sans GB", "Microsoft YaHei", "微软雅黑", Arial, sans-serif;
        -webkit-font-smoothing: antialiased;
        -moz-osx-font-smoothing: grayscale;
        text-align: center;
        color: #2c3e50;
        padding-top: 30px;
    }

    #app-footer a {
        padding-left: 0.2em;
        padding-right: 0.2em;
    }

    #app-footer {
        text-align: center;
        font-size: small;
    }

    .el-upload-dragger {
        width: 80vw !important;
    }

    #app-control {
        padding-top: 1em;
        padding-bottom: 1em;
    }


</style>
