<template>
    <my-page class="page-home" :title="title" :page="page">
        <div id="main">
            <ul class="book-list">
                <li class="item" v-for="book in books">
                    <router-link class="link" :to="'/books/' + book.id" :title="book.name">
                        <!-- <a class="remove" href="#" @click.stop.prevent="download(book)">下载</a> -->
                        <img class="cover" :src="book.cover" v-if="book.cover" />
                        <div class="cover-text" v-if="!book.cover">没有封面</div>
                    </router-link>
                    <div class="name">{{ book.name }}</div>
                    <a class="remove" href="#" @click.stop.prevent="remove(book)">删除</a>
                    <!-- <div class="author">{{ book.author }}</div> -->
                </li>
            </ul>
            <div class="empty-box" v-if="!books.length">
                <img src="/static/img/empty.svg"/>
                <div class="text">暂无书籍，你可以添加右上角的“+”添加书籍</div>
            </div>
            <ui-raised-button class="file-btn" label="从文件中导入" style="display: none">
                <input type="file" id="file" class="ui-file-button" @change="fileChange($event, 1)">
            </ui-raised-button>
        </div>
    </my-page>
</template>

<script>
    /* eslint-disable2 */
    import bookDb from '../util/bookDb2'
    import reader from '../util/reader'
    import {getCoverURL} from '../util/bookUtil'
    const ePub = window.ePub

    export default {
        data () {
            return {
                reader: null,
                title: 'epub 阅读器',
                books: [
                    {
                        id: '1',
                        name: '呵呵哒',
                        author: '陈建行',
                        content: '123'
                    },
                    {
                        id: '2',
                        name: '致我们即将逝去的青春',
                        author: '陈建行',
                        content: '123'
                    }
                ],
                page: {
                    menu: [
                        {
                            type: 'icon',
                            icon: 'add',
                            click: this.add,
                            title: '添加本地书籍'
                        },
                        {
                            type: 'icon',
                            icon: 'cloud_download',
                            click: this.addLink,
                            title: '添加云端书籍'
                        }
                    ]
                }
            }
        },
        mounted() {
            this.init()
        },
        destroyed() {
        },
        methods: {
            add() {
                document.getElementById('file').click()
            },
            init() {
                this.reader = reader
                console.log('初始化')
                bookDb.init(() => {
                    bookDb.getBooks(data => {
                        console.log('获取所有书籍')
                        console.log(data)
                        this.books = data
                        // 添加示例图书
                        let isInit = this.$storage.get('init', false)
                        if (!isInit) {
                            this.$storage.set('init', true)
                            this.addBookFromUrl('https://img1.yunser.com/epub/test.epub')
                        }
                    })
                })
            },
            addLink() {
                let link = prompt('请输入链接')
                console.log(link)
                this.addBookFromUrl(link)
            },
            addBookFromUrl(url) {
                this.book = ePub(url, {
                    width: 400,
                    height: 600,
                    spreads: false,
                    restore: true
                })
                this.book.getMetadata().then(meta => {
                    console.log('getMetadata')
                    console.log(meta)
                    bookDb.init(() => {
                        bookDb.addBook({
                            type: 'link',
                            id: '' + new Date().getTime(),
                            name: meta.bookTitle,
                            author: meta.creator,
                            content: url
                        }, () => {
                            bookDb.getBooks(data => {
                                console.log('获取所有书籍')
                                this.books = data
                                console.log(data)
                            })
                        })
                    })
                })
            },
            remove(book) {
                bookDb.init(() => {
                    bookDb.deleteBook(book.id, data => {
                        bookDb.getBooks(data => {
                            console.log('获取所有书籍')
                            console.log(data)
                            this.books = data
                        })
                    })
                })
            },
            download(book) {
                let fileName = 'dl.epub'
                let blob = book.content
                if (window.navigator.msSaveOrOpenBlob) {
                    navigator.msSaveBlob(blob, fileName)
                } else {
                    var link = document.createElement('a')
                    link.href = window.URL.createObjectURL(blob)
                    link.download = fileName
                    link.click()
                    window.URL.revokeObjectURL(link.href)
                }
            },
            fileChange(e) {
                let files = e.target.files
                if (!files.length) {
                    return
                }
                let file = e.target.files[0]
//                if (left === 1) {
//                    var f_name = file.name
//                    var f_type = f_name.substring(f_name.lastIndexOf("."))
//                }
                console.log(file.name)
                let reader = new FileReader()
                reader.onload = e => {
                    let content = e.target.result
                    this.book = ePub({
                        bookPath: content,
                        restore: false
                    })
                    this.book.getMetadata().then(meta => {
                        this.meta = meta
                        console.log('getMetadata')
                        console.log(meta)
                        getCoverURL(this.book, cover => {
                            console.log('首页封面')
                            console.log(cover)
                            bookDb.init(() => {
                                bookDb.addBook({
                                    id: '' + new Date().getTime(),
                                    name: meta.bookTitle,
                                    author: meta.creator,
                                    content: content,
                                    cover: cover
                                }, () => {
                                    bookDb.getBooks(data => {
                                        console.log('获取所有书籍')
                                        this.books = data
                                        console.log(data)
                                    })
                                })
                            })
                        })
                    })
                }
                reader.readAsArrayBuffer(file)
            }
        }
    }
</script>

<style lang="scss" scoped>
    @import "../scss/var";

    @keyframes rotate {
        to { transform: rotate(360deg); }
    }

    .book-list{
        @include clearfix;
        .item {
            position: relative;
            float: left;
            width: 140px;
            height: 196px;
            margin-right: 16px;
            margin-bottom: 16px;
            background-color: #fff;
            /*background-image: url("/static/img/bg_1.jpg");*/
            box-shadow: 0 1px 6px rgba(0,0,0,.117647), 0 1px 4px rgba(0,0,0,.117647);
            &:hover {
                .remove {
                    display: block;
                }
            }
        }
        .link {
            display: block;
            height: 100%;
            // padding: 16px;
            margin-bottom: 16px;
            color: inherit;
        }
        .name {
            // font-weight: bold;
        }
        .cover {
            max-width: 100%;
        }
        .cover-text {
            line-height: 196px;
            text-align: center;
            color: #999;
        }
        .author {
            position: absolute;
            bottom: 16px;
            left: 16px;
            white-space: nowrap;
            overflow: hidden;
            text-overflow: ellipsis;
        }
        .remove {
            display: none;
        }
    }
    .empty-box {
        width: 480px;
        margin: 80px auto;
        padding: 32px;
        text-align: center;
        background-color: #fff;
        box-shadow: 0 0.5px 1px 0 rgba(0, 0, 0, .12);
        img {
            margin-bottom: 16px;
            width: 100px;
            animation: rotate 5s infinite linear;
        }
        .text {
            color: #999;
        }
    }
</style>
