---
title: iOS-WKWebView秒开优化方案
date: 2022-02-10 16:50:46
categories:
- iOS
tags:
- WKWebView秒开优化方案
keywords:
- WKWebView秒开优化方案
description:
images:
---

### 简要

#### 解决场景
后台支持使用 HTML 文本编辑器发表 HTML 格式的帖子，移动端要兼容 HTML 格式帖子展示，必然要采用 WebView 加载 HTML 格式帖子，以达到同样的呈现效果。
<!-- more -->
#### 存在的问题
WebView 内容加载速度较原生有肉眼可见的延迟，在进入帖子详情页时，WebView 内容还未加载完成，页面显示空白。WebView 内容高度无法快速获得，导致原生列表嵌入 WebView 的布局中会出现内容突然撑开的效果，对于用户浏览来说，感观不太好。

#### 解决思路
- 优化 WebView 的加载效率，关于 HTML 类型帖子内容，后端接口返回的是 HTML 字符串，其中比较影响加载速度是对于图片的加载，所以需考虑对图片进行缓存，当再次加载时可以直接取缓存的数据，不需要网络请求获取。但是，即使这样优化后，加载速度还是无法达到原生页面加载的效果。
- 建立 WebView 缓存池，把加载完成的 WebView 放入缓存池中并设置对应的映射标识，下次加载同一帖子时，根据标识直接取出 WebView 展示。为考虑内存占用量问题，不能无限制数量的缓存 WebView 对象，得限制最大缓存数量。
- WebView 缓存池也仅仅只是对于首次加载后的情况做处理，并且缓存池的数量还做了限制，有可能出现无法命中缓存情况。因此，首次加载前得进行预加载处理，在帖子列表页时，对于当前屏幕展示的帖子内容进行预加载处理并放入缓存池，在进入帖子详情页时，从缓存池中取出展示，这样可以达到原生页面加载的效果。

### 实现方案
#### WebView 资源缓存
##### URLScheme 设置
WKWebView 提供 WKURLSchemeHandler 机制拦截资源请求，需传入自定义的资源链接 URLScheme，不能是 http、https，必须是自己定义的。
```
// WebURLSchemeHandler
static let urlScheme = "hooimageprex"

let config = WKWebViewConfiguration()
config.setURLSchemeHandler(WebURLSchemeHandler(), forURLScheme: WebURLSchemeHandler.urlScheme)
let webView = WKWebView.init(frame: CGRect(x: 0, y: 0, width: kScreenWidth - 16, height: 200), configuration: config)
```
后端返回的 HTML 字符串中，图片地址的 URLScheme 是 https，因此，只能通过正则匹配方式获取 图片地址在字符串中的位置，然后逐一修改成 hooimageprex:// 。

##### WKURLSchemeHandler 资源请求拦截处理
使用 WKURLSchemeHandler，需实现两个协议方法
```
/// 资源请求开始
func webView(_ webView: WKWebView, start urlSchemeTask: WKURLSchemeTask)

/// 资源请求结束
func webView(_ webView: WKWebView, stop urlSchemeTask: WKURLSchemeTask) 
```
WebView 进行资源请求时，便会调用该方法，我们需将资源地址恢复成正常的 http 地址，以资源地址为 key 去获取资源缓存，若获得缓存，则将资源数据填充给 urlSchemeTask。若未获得缓存，则通过资源地址下载，下载成功后，将资源数据填充给 urlSchemeTask 并缓存。如下载失败，调用 didFailWithError 方法结束资源请求。
```
/// 资源请求开始
func webView(_ webView: WKWebView, start urlSchemeTask: WKURLSchemeTask) {
    urlSchemeTaskList[urlSchemeTask.description] = false

    let urlSchemeUrlString = urlSchemeTask.request.url?.absoluteString ?? ""
    guard urlSchemeUrlString.starts(with: HooWebURLSchemeHandler.urlScheme) else { return }
    let resourceUrlString = urlSchemeUrlString.replacingOccurrences(of: HooWebURLSchemeHandler.urlScheme, with: "https")
    /// 首先读取缓存
    ImageCache.default.retrieveImage(forKey: resourceUrlString) { [weak self] result in
        switch result {
            case let .success(cacheResult):
                logDebug("HooWebURLSchemeHandler: 命中缓存")
                if let requestUrl = urlSchemeTask.request.url, let cacheImage = cacheResult.image, let imageData = cacheImage.compressedData() {
                    self?.urlTaskCompletionStatusList[resourceUrlString] = true
                    logDebug("HooWebURLSchemeHandler: 命中缓存，填充")
                    if self?.urlSchemeTaskIsStop(urlSchemeTask: urlSchemeTask) ?? false == false {
                        let mimeType = self?.mimeType(forPathExtension: requestUrl.pathExtension)
                        let response:URLResponse = URLResponse.init(url: requestUrl, mimeType: mimeType, expectedContentLength: imageData.count, textEncodingName: nil)
                        urlSchemeTask.didReceive(response)
                        urlSchemeTask.didReceive(imageData)
                        urlSchemeTask.didFinish()
                    }
                } else {
                    logDebug("HooWebURLSchemeHandler: 未命中缓存，请求")
                    self?.needNetWorkRequestImageSource(urlSchemeTask: urlSchemeTask, urlString: resourceUrlString)
                }
                break
            case .failure(_):
                logDebug("HooWebURLSchemeHandler: 未命中缓存错误，请求")
                self?.needNetWorkRequestImageSource(urlSchemeTask: urlSchemeTask, urlString: resourceUrlString)
                break
        }
    }
}
```
缓存未命中情况下，根据资源地址下载资源填充到 urlSchemeTask 并缓存
```
/// 缓存未命中，需自行下载并置入缓存
private func needNetWorkRequestImageSource(urlSchemeTask: WKURLSchemeTask, urlString: String) {
    if let downURL = URL.safe(urlString: urlString) {
        logDebug("HooWebURLSchemeHandler: 资源请求")
        ImageDownloader.default.downloadImage(with: downURL, options: [.processor(WebPProcessor.default), .cacheSerializer(WebPSerializer.default)]) { [weak self] result in
            switch result {
                case let .success(imageLoadingResult):
                    logDebug("HooWebURLSchemeHandler: 资源请求成功")
                    if let requestUrl = urlSchemeTask.request.url {
                        self?.urlTaskCompletionStatusList[urlString] = true
                        logDebug("HooWebURLSchemeHandler: 资源请求成功，填充")
                        ImageCache.default.store(imageLoadingResult.image, forKey: urlString)
                        if self?.urlSchemeTaskIsStop(urlSchemeTask: urlSchemeTask) ?? false == false {
                            let mimeType = self?.mimeType(forPathExtension: requestUrl.pathExtension)
                            let imageData = imageLoadingResult.originalData
                            let response:URLResponse = URLResponse.init(url: requestUrl, mimeType: mimeType, expectedContentLength: imageData.count, textEncodingName: nil)
                            urlSchemeTask.didReceive(response)
                            urlSchemeTask.didReceive(imageData)
                            urlSchemeTask.didFinish()
                        }
                    } else {
                        self?.urlTaskCompletionStatusList[urlString] = false
                        logDebug("HooWebURLSchemeHandler: 资源请求异常")
                        if self?.urlSchemeTaskIsStop(urlSchemeTask: urlSchemeTask) ?? false == false {
                            urlSchemeTask.didFailWithError(HooWebURLSchemeHandlerDownError.downFail)
                        }
                    }
                    break
                default:
                    self?.urlTaskCompletionStatusList[urlString] = false
                    logDebug("HooWebURLSchemeHandler: 资源请求失败")
                    if self?.urlSchemeTaskIsStop(urlSchemeTask: urlSchemeTask) ?? false == false {
                        urlSchemeTask.didFailWithError(HooWebURLSchemeHandlerDownError.downFail)
                    }
                    break
            }
        }
    } else {
        urlTaskCompletionStatusList[urlString] = false
        logDebug("HooWebURLSchemeHandler: 资源请求链接错误")
        if urlSchemeTaskIsStop(urlSchemeTask: urlSchemeTask) == false {
            urlSchemeTask.didFailWithError(HooWebURLSchemeHandlerDownError.downFail)
        }
    }
}
```

当 WebView 释放时，urlSchemeTask 会停止任务并调用 "func webView(_ webView: WKWebView, stop urlSchemeTask: WKURLSchemeTask)" ，如果这时我们还往 urlSchemeTask 中填充资源数据，会引发程序崩溃。
```
/// 资源请求结束
func webView(_ webView: WKWebView, stop urlSchemeTask: WKURLSchemeTask) {
    urlSchemeTaskList[urlSchemeTask.description] = true
}
```
urlSchemeTaskList 用于记录 urlSchemeTask 是否停止任务，在上述关于 urlSchemeTask 填充资源数据之前，都会对 urlSchemeTask 的状态进行判断。
```
/// 判断资源请求是否已停止
private func urlSchemeTaskIsStop(urlSchemeTask: WKURLSchemeTask) -> Bool {
    return urlSchemeTaskList[urlSchemeTask.description] ?? false
}
```
#### HooWKWebViewPool 缓存池
创建缓存池单例
```
/// webView 缓存池
class HooWKWebViewPool {
    static let share = HooWKWebViewPool()
    /// identifiler 和 webView 的映射
    private var poolDict: [String: HooWKPrepareWebView] = [:]
    /// identifiler 数组
    private var identifilerList: [String] = []
    /// 最大缓存数量
    private let initialViewsMaxCount = 20
    /// 队列
    private let lockQueue = DispatchQueue(label: "WKWebViewPool_lock_queue", qos: .userInitiated)
    ......
```

根据 HTML 字符串和对应的缓存标识进行预加载处理，首先，通过缓存标识判断，该内容是否已经进行了预加载处理，若未进行过预加载处理，则创建新的 WebView 去加载 HTML 字符串，并存入缓存池中。
```
/// 预加载web
/// - Parameter infoList: 预加载信息列表
func prepare(infoList: [HooWKWebViewPrepareInfo]) {
    for info in infoList {
        logDebug("预加载web \(info.identifiler ?? "")")
        guard let identifiler = info.identifiler, identifiler.count > 0 else { continue }
        if let cacheWebView = getWebView(identifiler: identifiler) {
            if cacheWebView.isNeedReload {
                cacheWebView.reloadContent()
            }
            continue
        }
        let webView = createWebView()
        webView.loadContent(content: info.content ?? "")
        cachePrepareWebView(webView: webView, prepareInfo: info)
    }
}

/// 将 webView 存入缓存池
/// - Parameters:
/// - prepareInfo: 预加载信息
func cachePrepareWebView(webView: HooWKPrepareWebView, prepareInfo: HooWKWebViewPrepareInfo) {
    lockQueue.async(flags: .barrier) { [weak self] in
        let identifiler = prepareInfo.identifiler ?? ""
        if identifiler.count == 0 || self?.identifilerList.contains(identifiler) ?? false {
            return
        }
        if self?.identifilerList.count ?? 0 >= self?.initialViewsMaxCount ?? 0 {
            let removeIdentifiler = self?.identifilerList.removeFirst()
            self?.poolDict.removeValue(forKey: removeIdentifiler ?? "")
        }
        logDebug("存入预加载web \(identifiler)")
        self?.identifilerList.append(identifiler)
        self?.poolDict[identifiler] = webView
    }
}

/// 根据 identifiler 获取 webView
/// - Parameter identifiler: 标识
/// - Returns: webView
func getWebView(identifiler: String) -> HooWKPrepareWebView? {
    lockQueue.sync {
        return self.poolDict[identifiler]
    }
}
    
/// 创建预加载 webView
/// - Returns: webView
private func createWebView() -> HooWKPrepareWebView {
    return HooWKPrepareWebView()
}
```
### 总结
上述只是对于简单 HTML 字符串加载的优化处理，对于资源的缓存也只处理了图片资源，对于通过 url 加载网页的秒开优化，还需前后端共同配合优化。
