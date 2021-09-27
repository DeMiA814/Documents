## Tips

### createOfferについて
  　映像・音声の受信のみを行いたい場合は、offerの設定に追記をしなければならない。これは、WebRTCがデフォルトでは双方向受信を想定しており、offerに映像・音声の情報が含まれていないとうまくice candidateを探し始めてくれないからだ。具体的にはブラウザに搭載されているWebRTCだと
  ```
  {
      offerToReceiveAudio: true,
      offerToReceiveVideo: true
  }
  ```
こんな感じで、WebRTC-for-flutterだと
  ```
  {
    'mandatory': {
      'OfferToReceiveAudio': true,
      'OfferToReceiveVideo': true,
    },
    'optional': [],
  }
  ```
こんなかんじ。

### RTCRtpTransceiver
　先ほどの話の補足になるが、offerの時に指定する方法以外にもあるらしい（もしくは本来両方設定するのかもしれない。ayameは設定していた）。それにはまず、
[RTCRtpTransceiver](https://developer.mozilla.org/en-US/docs/Web/API/RTCRtpTransceiver)
という技術を理解する必要がある。これは、簡単にいうとペアリングとメディア情報の一部を管理しているものなのだが、一方向通信の際はこのペアリングにsend only/recieve onlyが存在することを伝える必要がある（先ほども書いたが、WebRTCはデフォルトでは双方向通信を想定しているため）。そしてもう一つ重要な点は、このRTCRtpTransceiverはいつもは自動で作成されているという点だ。普段mediaStreamをaddTrackしていると思うが、この際に内部的に双方向通信を指定してaddTransceiverしている。つまり、一方向通信がしたければ、addTrackの代わりにaddTranscieverを手動で呼び出し、一方公通信であることを伝えれば良い。具体的には
```
pc.addTransceiver('audio', { direction: 'recvonly' });
pc.addTransceiver('video', { direction: 'recvonly' });
```
こんな感じだ。詳しい内容は
[https://lealog.hateblo.jp/entry/2019/03/12/114529](https://lealog.hateblo.jp/entry/2019/03/12/114529)
がかなり分かりやすくまとめてくれており、参考になる。

### SDPのPlan A(Unified Plan)とPlan b
  　SDPの記述方法には、Plan AとPlan Bという物が存在する。WebRTC発足当時は、両方の記述方法が混在していたが、最近ではPlan AをUnified Planと称し、Plan Bを無くす方向らしい。詳しい内容は
  [https://www.isoroot.jp/blog/2211/](https://www.isoroot.jp/blog/2211/)
  がかなりまとめてくれており、参考になる。

### WebRTCの活用について
　ここまでWebRTCを勉強して感じたことだが、WebRTCという技術は非常に面倒臭いが良い立ち位置にいると思う。というのも、クライアントの最大数、双方向か片方向かで大分採用すべき方式が変化するが、かなり広く対応できるからだ。

　まず、WebRTCとWebRTC DataChannelの違いだが、前者は映像/音声を簡単に送りたい場合、後者はそれ以外で活躍できる。WebRTCは映像/音声のみだと思われがちだが、DataChannelの登場により改善された。DataChannelの概要は[こちら](https://voluntas.medium.com/webrtc-sfu-%E3%81%A8-datachannel-%E3%81%AE%E8%AA%B2%E9%A1%8C-bf254653f61e)。そしてこれも重要なことだが、WebRTCはDTLS-SRTPというRFCに規定されている仕組みにしか対応しておらず、全く融通が効かない。この仕組みがなかなかに良くなく、後述のSFUにおいては苦しめられることになる。よって、映像/音声を利用する場合でも、色々とフォーマットをいじりたい場合などはDataChannelを活用するべきであろう。

　そして次に、P2Pを採用するか否かである。P2Pには大きなメリットがある。サーバーが必要ないため、費用が浮きリアルタイム性も高い。しかし同時にデメリットもある。人数が増えれば増えるほど処理が重くなり、限界がある。そこで採用されているのが、SFUとMCUだ。と言ってもMCUは既にほとんど廃れており、SFUが主流となっている。SFUの詳しい解説は[こちら](https://voluntas.medium.com/webrtc-sfu-%E3%81%A8-datachannel-%E3%81%AE%E8%AA%B2%E9%A1%8C-bf254653f61e)。また、SFUのメリットは他にもある。サーバーを介するため、録画の保存やプレイバックなどができることだ。ただし、SFUを使う際にはWebRTC DataChannelを用いないと、非常に面倒臭いことになる。SFUサーバーにおいてDTLS-SRTPの復号化/暗号化が必要なのだ。そのため、SFUではDataChannelが用いられることが多い。

　ではどこが境界線なのだろうか。今回お世話になった[時雨堂様によると](https://sora.shiguredo.jp/knowledge)、3名までのクライアント間での通信ならば、P2Pも検討するそうだ。意外と少ない。テストを行なっていないのでわからないが、体感では1対nではn側が4名が限界に感じた。どちらにせよ、WebRTC+P2Pの組み合わせは非常に限られたものということだ。
