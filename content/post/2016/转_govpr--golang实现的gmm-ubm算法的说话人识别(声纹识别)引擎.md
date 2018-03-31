
---
date: 2016-12-31T11:32:55+08:00
title: "govpr--golang实现的gmm-ubm算法的说话人识别(声纹识别)引擎"
description: ""
disqus_identifier: 1485833575777871734
slug: "govpr--golangshi-xian-de-gmm-ubmsuan-fa-de-shui-hua-ren-shi-bie-(sheng-wen-shi-bie-)yin-qing"
source: "https://segmentfault.com/a/1190000007382096"
tags: 
- golang 
categories:
- 编程语言与开发
---

简介
----

govpr是golang 实现的基于 GMM-UBM
说话人识别引擎(声纹识别),可用于语音验证,身份识别的场景.\
目前暂时仅支持汉语数字的语音,语音格式为wav格式(比特率16000,16bits,单声道)

安装
----

go get github.com/liuxp0827/govpr

示例
----

如下是一个简单的示例. 可跳转至
[example](https://github.com/liuxp0827/govpr/blob/master/example)\
查看详细的例子,示例中的语音为纯数字8位数字.语音验证后得到一个得分,可设置阈值来判断验证语音是否为注册训练者本人.

    package main

    import (
        "github.com/liuxp0827/govpr"
        "github.com/liuxp0827/govpr/log"
        "github.com/liuxp0827/govpr/waveIO"
        "io/ioutil"
    )

    type engine struct {
        vprEngine *govpr.VPREngine
    }

    func NewEngine(sampleRate, delSilRange int, ubmFile, userModelFile string) *engine {
        return &engine{
            vprEngine: govpr.NewVPREngine(sampleRate, delSilRange, ubmFile, userModelFile),
        }
    }

    func (this *engine) DestroyEngine() {
        this.vprEngine = nil
    }

    func (this *engine) TrainSpeech(buffers [][]byte) error {

        var err error
        count := len(buffers)
        for i := 0; i < count; i++ {
            err = this.vprEngine.AddTrainBuffer(buffers[i])
            if err != nil {
                log.Error(err)
                return err
            }
        }

        defer this.vprEngine.ClearTrainBuffer()
        defer this.vprEngine.ClearAllBuffer()

        err = this.vprEngine.TrainModel()
        if err != nil {
            log.Error(err)
            return err
        }

        return nil
    }

    func (this *engine) RecSpeech(buffer []byte) error {

        err := this.vprEngine.AddVerifyBuffer(buffer)
        defer this.vprEngine.ClearVerifyBuffer()
        if err != nil {
            log.Error(err)
            return err
        }

        err = this.vprEngine.VerifyModel()
        if err != nil {
            log.Error(err)
            return err
        }

        Score := this.vprEngine.GetScore()
        log.Infof("vpr score: %f", Score)
        return nil
    }

    func main() {
        log.SetLevel(log.LevelDebug)

        vprEngine := NewEngine(16000, 50, "../ubm/ubm", "model/test.dat")
        trainlist := []string{
            "wav/train/01_32468975.wav",
            "wav/train/02_58769423.wav",
            "wav/train/03_59682734.wav",
            "wav/train/04_64958273.wav",
            "wav/train/05_65432978.wav",
        }

        trainBuffer := make([][]byte, 0)

        for _, file := range trainlist {
            buf, err := loadWaveData(file)
            if err != nil {
                log.Error(err)
                return
            }
            trainBuffer = append(trainBuffer, buf)
        }

        verifyBuffer, err := waveIO.WaveLoad("wav/verify/34986527.wav")
        if err != nil {
            log.Error(err)
            return
        }

        vprEngine.TrainSpeech(trainBuffer)
        vprEngine.RecSpeech(verifyBuffer)
    }

    func loadWaveData(file string) ([]byte, error) {
        data, err := ioutil.ReadFile(file)
        if err != nil {
            return nil, err
        }
        // remove .wav header info 44 bits
        data = data[44:]
        return data, nil
    }

