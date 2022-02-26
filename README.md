# 西瓜视频下载脚本

打开西瓜视频网站的某个列表页，例如 [宠物](https://www.ixigua.com/channel/chongwu) 列表页，
然后按 `F12` 打开**开发者工具**，切换到**控制台**面板，将以下代码粘贴进去，再按 `Enter` 回车键执行即可。

```javascript
async function main() {
    let container = document.querySelector('.FeedContainer__content')
    let nodeList = container.querySelectorAll('a.HorizontalFeedCard__coverWrapper')

    for (let element of nodeList) {
        let title = element.getAttribute('title')
        let url = location.origin + element.getAttribute('href')

        let videoUrl = await retryWrapper(getVideoUrl, 5)(url)

        let blob = await retryWrapper(download, 10)(title, videoUrl)
        let link = URL.createObjectURL(blob)
        Object.assign(document.createElement('a'), { href: link, download: `${title}.mp4` }).click()

        await new Promise(resolve => setTimeout(resolve, 5000))
    }
}

async function getVideoUrl(url) {
    let iframe = document.createElement('iframe')

    document.body.appendChild(iframe)

    try {
        await new Promise((resolve, reject) => Object.assign(iframe, { onload: resolve, onerror: reject, src: url }))

        let { video_list: videoList } = iframe.contentWindow._SSR_HYDRATED_DATA.anyVideo.gidInformation.packerData.video.videoResource.normal
        let key = Object.keys(videoList).sort().pop()
        let { main_url: videoUrl } = videoList[key]

        return videoUrl
    } finally {
        document.body.removeChild(iframe)
    }
}

async function download(title, url) {
    let board = document.createElement('div')
    board.style = `
        position: fixed; left: 0; top: 0; z-index: 10000; background: lightgray;
        font-size: 20px; line-height: 40px; padding: 18px 2em; box-shadow: #b6b6b6 0 0 20px 14px;
        white-space: pre; text-align: center;
    `
    document.body.appendChild(board)

    try {
        let xhr = new XMLHttpRequest()
        xhr.open('GET', url)
        xhr.responseType = 'arraybuffer'
        xhr.onprogress = ({ loaded, total }) => board.textContent = `文件：${title}\n进度：${(loaded / total * 100).toFixed(1)}%`
        await new Promise((resolve, reject) => (Object.assign(xhr, { onload: resolve, onerror: reject }), xhr.send()))

        const { response } = xhr
        return new Blob([response], {type: 'arraybuffer'})
    } finally {
        document.body.removeChild(board)
    }
}

function retryWrapper(fn, times = 3) {
    return async function() {
        for (let i = 0; i < times; i++) {
            try {
                return await fn.apply(this, arguments)
            } catch (e) {
                await new Promise(resolve => setTimeout(resolve, 500 * Math.pow(2, i)))
            }
        }
    }
}

main()
```
