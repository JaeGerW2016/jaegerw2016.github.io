---
title: 使用slack来接收AlertManager的告警信息
date: 2019-05-28 23:47:30
tags:
---
在国内，大部分公司的员工目前在办公时用的沟通工具仍是邮件、QQ或者微信，文件会放到共享文件夹，或者用邮件、即时通讯工具来传输。而邮件使用起来比较费劲，没有即时性，而QQ和微信会让我们把生活与工作混在一起，降低做事效率，很多人在工作的时候还经常刷朋友圈、看QQ空间，这都是不好的现象。国外的现象跟国内也是大同小异。

Slack较早的发现了这个现象，瞄准了企业内部沟通协作这一空白点，做出了一款提高企业员工内部协作效率的软件。它将工作中所有信息的接口统统整合在一起，从而提高内部协作的效率，提高信息的利用率。

在Slack的界面上，我们可以看到它中间部分为谈话内容，和一般的通讯工具类似，而在右侧你可以随时查找在对话过程中涉及到的文件，目前Slack已经整合了包括Google Drive、Google+ Hangouts、Twitter、Dropbox、Asana、GitHub、RSS、Trello等各类通讯服务、团队协作服务、开发工具。

#### 创建一个工作空间

![](https://i.loli.net/2019/05/29/5ceddf52d3dee33759.png)

![](https://i.loli.net/2019/05/29/5cede0d463b8726572.png)



#### 新增频道

![](https://i.loli.net/2019/05/29/5cede197007f943685.png)

#### 新增Incoming WebHooks 

![](https://i.loli.net/2019/05/29/5cede1da6b60c77557.png)

![](https://i.loli.net/2019/05/29/5cede27d8a53e62097.png)

####  配置Incoming WebHooks

![](https://i.loli.net/2019/05/29/5cedf86f93a9380767.png)

![](https://i.loli.net/2019/05/29/5cedf92194b6465316.png)

![](https://i.loli.net/2019/05/29/5cedf98537af477028.png)



### AlertManager Secret

`alertmanager.yaml`

```yaml
global:
  resolve_timeout: 5m
  smtp_smarthost: "smtp.163.com:25"
  smtp_from: "xxx@163.com" #edit your own 163 smtp config
  smtp_auth_username: "xxx"
  smtp_auth_password: "xxx"
route:
  group_by: 
  - "alertname"
  - "cluster"
  - "service"
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 15m
  "receiver": "slack_webhook"
  routes:
  - match_re:
      "alertname": ^(host_cpu_usage|node_filesystem_free|host_down|TargetDown)$
    receiver: "slack_webhook"
  - match:
      severity: critical
    receiver: "slack_webhook"
  - match:
      severity: warning
    receiver: "slack_webhook"
  - match:
      "alertname": DeadMansSwitch
    receiver: "null"
receivers:
- name: "null"
- name: "team-X-mails"
  email_configs:
  - to: "needtochange@admin.com" 
- name: slack_webhook
  slack_configs:
  - send_resolved: false
    api_url: https://hooks.slack.com/services/NEEDTOCHANGE/xxxxxxxxxxxxx
    channel: prometheus
    username: '{{ template "slack.default.username" . }}'
    color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
    title: '{{ template "slack.default.title" . }}'
    title_link: '{{ template "slack.default.titlelink" . }}'
    pretext: '{{ .CommonAnnotations.summary }}'
    text: |-
      {{ range .Alerts }}
         *Alert:* {{ .Annotations.summary }} - `{{ .Labels.severity }}`
        *Description:* {{ .Annotations.description }}
        *Details:*
        {{ range .Labels.SortedPairs }} • *{{ .Name }}:* `{{ .Value }}`
        {{ end }}
      {{ end }}
    fallback: '{{ template "slack.default.fallback" . }}'
    icon_emoji: '{{ template "slack.default.iconemoji" . }}'
    icon_url: '{{ template "slack.default.iconurl" . }}'
templates: 
- '/etc/alertmanager/config/*.tmpl'
```

`default.tmpl`

```
{{ define "__alertmanager" }}Cluster: CLUSTER_NAME{{ end }}
{{ define "__alertmanagerURL" }}{{ .ExternalURL }}/#/alerts?receiver={{ .Receiver }}{{ end }}

{{ define "__subject" }}[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .GroupLabels.SortedPairs.Values | join " " }} {{ if gt (len .CommonLabels) (len .GroupLabels) }}({{ with .CommonLabels.Remove .GroupLabels.Names }}{{ .Values | join " " }}{{ end }}){{ end }}{{ end }}
{{ define "__description" }}{{ end }}

{{ define "__text_alert_list" }}{{ range . }}Labels:
{{ range .Labels.SortedPairs }} - {{ .Name }} = {{ .Value }}
{{ end }}Annotations:
{{ range .Annotations.SortedPairs }} - {{ .Name }} = {{ .Value }}
{{ end }}Source: {{ .GeneratorURL }}
{{ end }}{{ end }}


{{ define "slack.default.title" }}{{ template "__subject" . }}{{ end }}
{{ define "slack.default.username" }}{{ template "__alertmanager" . }}{{ end }}
{{ define "slack.default.fallback" }}{{ template "slack.default.title" . }} | {{ template "slack.default.titlelink" . }}{{ end }}
{{ define "slack.default.pretext" }}{{ end }}
{{ define "slack.default.titlelink" }}{{ template "__alertmanagerURL" . }}{{ end }}
{{ define "slack.default.iconemoji" }}{{ end }}
{{ define "slack.default.iconurl" }}{{ end }}
{{ define "slack.default.text" }}{{ end }}
{{ define "slack.default.footer" }}{{ end }}
```

`alertmanager-secret-slack.yaml`

```yaml
apiVersion: v1
data:
  alertmanager.yaml: 
Z2xvYmFsOgogIHJlc29sdmVfdGltZW91dDogNW0KICBzbXRwX3NtYXJ0aG9zdDogInNtdHAuMTYzLmNvbToyNSIKICBzbXRwX2Zyb206ICJ4eHhAMTYzLmNvbSIgI2VkaXQgeW91ciBvd24gMTYzIHNtdHAgY29uZmlnCiAgc210cF9hdXRoX3VzZXJuYW1lOiAieHh4IgogIHNtdHBfYXV0aF9wYXNzd29yZDogInh4eCIKcm91dGU6CiAgZ3JvdXBfYnk6IAogIC0gImFsZXJ0bmFtZSIKICAtICJjbHVzdGVyIgogIC0gInNlcnZpY2UiCiAgZ3JvdXBfd2FpdDogMzBzCiAgZ3JvdXBfaW50ZXJ2YWw6IDVtCiAgcmVwZWF0X2ludGVydmFsOiAxNW0KICAicmVjZWl2ZXIiOiAic2xhY2tfd2ViaG9vayIKICByb3V0ZXM6CiAgLSBtYXRjaF9yZToKICAgICAgImFsZXJ0bmFtZSI6IF4oaG9zdF9jcHVfdXNhZ2V8bm9kZV9maWxlc3lzdGVtX2ZyZWV8aG9zdF9kb3dufFRhcmdldERvd24pJAogICAgcmVjZWl2ZXI6ICJzbGFja193ZWJob29rIgogIC0gbWF0Y2g6CiAgICAgIHNldmVyaXR5OiBjcml0aWNhbAogICAgcmVjZWl2ZXI6ICJzbGFja193ZWJob29rIgogIC0gbWF0Y2g6CiAgICAgIHNldmVyaXR5OiB3YXJuaW5nCiAgICByZWNlaXZlcjogInNsYWNrX3dlYmhvb2siCiAgLSBtYXRjaDoKICAgICAgImFsZXJ0bmFtZSI6IERlYWRNYW5zU3dpdGNoCiAgICByZWNlaXZlcjogIm51bGwiCnJlY2VpdmVyczoKLSBuYW1lOiAibnVsbCIKLSBuYW1lOiAidGVhbS1YLW1haWxzIgogIGVtYWlsX2NvbmZpZ3M6CiAgLSB0bzogIm5lZWR0b2NoYW5nZUBhZG1pbi5jb20iIAotIG5hbWU6IHNsYWNrX3dlYmhvb2sKICBzbGFja19jb25maWdzOgogIC0gc2VuZF9yZXNvbHZlZDogZmFsc2UKICAgIGFwaV91cmw6IGh0dHBzOi8vaG9va3Muc2xhY2suY29tL3NlcnZpY2VzL3h4eHgveHh4eHh4eHh4eHh4eAogICAgY2hhbm5lbDogYWxlcnRtYW5hZ2VyCiAgICB1c2VybmFtZTogJ3t7IHRlbXBsYXRlICJzbGFjay5kZWZhdWx0LnVzZXJuYW1lIiAuIH19JwogICAgY29sb3I6ICd7eyBpZiBlcSAuU3RhdHVzICJmaXJpbmciIH19ZGFuZ2Vye3sgZWxzZSB9fWdvb2R7eyBlbmQgfX0nCiAgICB0aXRsZTogJ3t7IHRlbXBsYXRlICJzbGFjay5kZWZhdWx0LnRpdGxlIiAuIH19JwogICAgdGl0bGVfbGluazogJ3t7IHRlbXBsYXRlICJzbGFjay5kZWZhdWx0LnRpdGxlbGluayIgLiB9fScKICAgIHByZXRleHQ6ICd7eyAuQ29tbW9uQW5ub3RhdGlvbnMuc3VtbWFyeSB9fScKICAgIHRleHQ6IHwtCiAgICAgIHt7IHJhbmdlIC5BbGVydHMgfX0KICAgICAgICAgKkFsZXJ0Oioge3sgLkFubm90YXRpb25zLnN1bW1hcnkgfX0gLSBge3sgLkxhYmVscy5zZXZlcml0eSB9fWAKICAgICAgICAqRGVzY3JpcHRpb246KiB7eyAuQW5ub3RhdGlvbnMuZGVzY3JpcHRpb24gfX0KICAgICAgICAqRGV0YWlsczoqCiAgICAgICAge3sgcmFuZ2UgLkxhYmVscy5Tb3J0ZWRQYWlycyB9fSDigKIgKnt7IC5OYW1lIH19OiogYHt7IC5WYWx1ZSB9fWAKICAgICAgICB7eyBlbmQgfX0KICAgICAge3sgZW5kIH19CiAgICBmYWxsYmFjazogJ3t7IHRlbXBsYXRlICJzbGFjay5kZWZhdWx0LmZhbGxiYWNrIiAuIH19JwogICAgaWNvbl9lbW9qaTogJ3t7IHRlbXBsYXRlICJzbGFjay5kZWZhdWx0Lmljb25lbW9qaSIgLiB9fScKICAgIGljb25fdXJsOiAne3sgdGVtcGxhdGUgInNsYWNrLmRlZmF1bHQuaWNvbnVybCIgLiB9fScKdGVtcGxhdGVzOiAKLSAnL2V0Yy9hbGVydG1hbmFnZXIvY29uZmlnLyoudG1wbCc=
  default.tmpl: e3sgZGVmaW5lICJfX2FsZXJ0bWFuYWdlciIgfX1DbHVzdGVyOiBDTFVTVEVSX05BTUV7eyBlbmQgfX0Ke3sgZGVmaW5lICJfX2FsZXJ0bWFuYWdlclVSTCIgfX17eyAuRXh0ZXJuYWxVUkwgfX0vIy9hbGVydHM/cmVjZWl2ZXI9e3sgLlJlY2VpdmVyIH19e3sgZW5kIH19Cgp7eyBkZWZpbmUgIl9fc3ViamVjdCIgfX1be3sgLlN0YXR1cyB8IHRvVXBwZXIgfX17eyBpZiBlcSAuU3RhdHVzICJmaXJpbmciIH19Ont7IC5BbGVydHMuRmlyaW5nIHwgbGVuIH19e3sgZW5kIH19XSB7eyAuR3JvdXBMYWJlbHMuU29ydGVkUGFpcnMuVmFsdWVzIHwgam9pbiAiICIgfX0ge3sgaWYgZ3QgKGxlbiAuQ29tbW9uTGFiZWxzKSAobGVuIC5Hcm91cExhYmVscykgfX0oe3sgd2l0aCAuQ29tbW9uTGFiZWxzLlJlbW92ZSAuR3JvdXBMYWJlbHMuTmFtZXMgfX17eyAuVmFsdWVzIHwgam9pbiAiICIgfX17eyBlbmQgfX0pe3sgZW5kIH19e3sgZW5kIH19Cnt7IGRlZmluZSAiX19kZXNjcmlwdGlvbiIgfX17eyBlbmQgfX0KCnt7IGRlZmluZSAiX190ZXh0X2FsZXJ0X2xpc3QiIH19e3sgcmFuZ2UgLiB9fUxhYmVsczoKe3sgcmFuZ2UgLkxhYmVscy5Tb3J0ZWRQYWlycyB9fSAtIHt7IC5OYW1lIH19ID0ge3sgLlZhbHVlIH19Cnt7IGVuZCB9fUFubm90YXRpb25zOgp7eyByYW5nZSAuQW5ub3RhdGlvbnMuU29ydGVkUGFpcnMgfX0gLSB7eyAuTmFtZSB9fSA9IHt7IC5WYWx1ZSB9fQp7eyBlbmQgfX1Tb3VyY2U6IHt7IC5HZW5lcmF0b3JVUkwgfX0Ke3sgZW5kIH19e3sgZW5kIH19CgoKe3sgZGVmaW5lICJzbGFjay5kZWZhdWx0LnRpdGxlIiB9fXt7IHRlbXBsYXRlICJfX3N1YmplY3QiIC4gfX17eyBlbmQgfX0Ke3sgZGVmaW5lICJzbGFjay5kZWZhdWx0LnVzZXJuYW1lIiB9fXt7IHRlbXBsYXRlICJfX2FsZXJ0bWFuYWdlciIgLiB9fXt7IGVuZCB9fQp7eyBkZWZpbmUgInNsYWNrLmRlZmF1bHQuZmFsbGJhY2siIH19e3sgdGVtcGxhdGUgInNsYWNrLmRlZmF1bHQudGl0bGUiIC4gfX0gfCB7eyB0ZW1wbGF0ZSAic2xhY2suZGVmYXVsdC50aXRsZWxpbmsiIC4gfX17eyBlbmQgfX0Ke3sgZGVmaW5lICJzbGFjay5kZWZhdWx0LnByZXRleHQiIH19e3sgZW5kIH19Cnt7IGRlZmluZSAic2xhY2suZGVmYXVsdC50aXRsZWxpbmsiIH19e3sgdGVtcGxhdGUgIl9fYWxlcnRtYW5hZ2VyVVJMIiAuIH19e3sgZW5kIH19Cnt7IGRlZmluZSAic2xhY2suZGVmYXVsdC5pY29uZW1vamkiIH19e3sgZW5kIH19Cnt7IGRlZmluZSAic2xhY2suZGVmYXVsdC5pY29udXJsIiB9fXt7IGVuZCB9fQp7eyBkZWZpbmUgInNsYWNrLmRlZmF1bHQudGV4dCIgfX17eyBlbmQgfX0Ke3sgZGVmaW5lICJzbGFjay5kZWZhdWx0LmZvb3RlciIgfX17eyBlbmQgfX0=
kind: Secret
metadata:
  name: alertmanager-main
  namespace: monitoring
type: Opaque
```



#### 结果

![](https://i.loli.net/2019/05/29/5cedfe3eb519386802.png)

#### 总结
如你所见，告警信息里包含很多详细信息，可以第一时间获取到，至于其及时性与是否具有滞后性得在实际使用过程中慢慢调整，你可以随意调整Prometheus的alert在注释块中包含标识符和描述标记以及自定义使用的格式default.tmpl
