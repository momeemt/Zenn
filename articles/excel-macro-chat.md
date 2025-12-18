---
title: "Microsoft Excel上で動作するチャットアプリケーションを作っていた話"
emoji: "👨‍⚕️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Excel, VisualBasic, VB, Office, Microsoft]
published: true
published_at: 2020-12-16 21:00
---

:::message
この記事は、筆者が過去に書いた記事を Zenn にアップロードしたものです。
:::

# はじめに
こんにちは。高校2年の[momeemt](https://www.twitter.com/momeemt)です。

今回は、高校1年生の時に「情報の科学」という単位の授業中に合間を縫って開発していた、**Exchat** というアプリケーションについて紹介します。

## 開発動機
学校のパソコンでは他のパソコンと通信することもできないので、授業内容に関する知見や疑問などについて共有することはできません。
基本的に、授業ではExcelの使い方などを学んでいたので、Excel上で他のパソコンとメッセージをやり取りすることができれば、とても便利だなと感じました。
Exchatは、Microsoft Excelで動作する **Visual Basic** を用いて、データベースはインストールされていなかった（し、Microsoft Accessのことを知らなかった）ので、テキスト形式でメッセージを保存して、LANに保存する形式を取りました。

## メッセージの作成
VBAでは、フォームを簡単に利用できるので、入力フォームから受け取ったメッセージが空でなければ、LANに新しく作成します。

```basic:メッセージの作成
Sub message_submit()
    Dim textbox_data As String
    textbox_data = UserForm4.TextBox1.Value
    If textbox_data = "" Then
        textbox_data = UserForm1.TextBox1.Value
    End If
    If Not textbox_data = "" Then
        Dim target_path As String: target_path = ROOT_APP_PATH & "models\message_model\public\"
        target_path = target_path & get_current_number_of_files(target_path) + 1 & ".pasta"

        Dim message_data As String
        message_data = textbox_data

        Dim userid As String: userid = init.user_id

        Open target_path For Output As #1
            Print #1, easySecure.easy_secure(textbox_data)
            Print #1, easySecure.easy_secure(Now)
            Print #1, easySecure.easy_secure(userid)
        Close #1

        UserForm1.TextBox1.Value = ""
        UserForm4.TextBox1.Value = ""
    End If
End Sub

Function get_current_number_of_files(target_path As String)
    Dim index As Long: index = 1
    While Not Dir(target_path & index & ".pasta") = ""
        index = index + 1
    Wend
    get_current_number_of_files = index - 1
End Function
```

冷静になると、UserFormにも命名をしないと恐ろしく読みづらいのですが、それに気づかずに毎回困っていました。
ここで、`easySecure`についてですが、LANにチャット情報をそのまま保存するのは困るなと思い、

```basic:XOR暗号化
Function easy_secure(ByVal str As String)
    Dim password As String: password = "securepass"

    Do While Len(str) > Len(password)
        password = password & password
    Loop
    password = Left(password, Len(str))

    Dim secured As String

    For i = 1 To Len(str)
        secured = secured & CStr(Asc(Mid(str, i, 1)) Xor Asc(Mid(password, i, 1))) & " "
    Next

    easy_secure = secured
End Function

Function easy_r_secure(ByVal str As String)
    Dim password As String: password = "securepass"

    Do While Len(str) > Len(password)
        password = password & password
    Loop
    password = Left(password, Len(str))

    Dim secured_str_arr() As String
    Dim raw_string As String

    secured_str_arr = Split(str, " ")

    For j = LBound(secured_str_arr) To UBound(secured_str_arr) - 1
        raw_string = raw_string & Chr(CLng(secured_str_arr(j)) Xor Asc(Mid(password, j + 1, 1)))
    Next j

    easy_r_secure = raw_string
End Function
```

とても簡単な文字列XOR暗号化をかけていました。
このままではいけないという危機感があったのか、RSA暗号を実装しかけている痕跡が残っていました。

```basic:RSA暗号実装の途中
Function encryption(raw_message As String)
    ' 文字をshift_jisコードに
    Dim shift_jis__array
    ReDim shift_jis_array(Len(raw_message))
    For i = 1 To Len(raw_message)
        shift_jis_array(i) = Asc(Mid$(raw_message, i, 1))
        MsgBox Asc(Mid$(raw_message, i, 1))
    Next


End Function


Function decryption()

End Function

Sub generate_public_key()
    Dim n As Double
    Dim public_key_dir As String

    public_key_dir = ROOT_APP_PATH & "keys\"
    n = generate_prime_number() * generate_prime_number()

    Dim WshNetworkObject As Object
    Set WshNetworkObject = CreateObject("WScript.Network")
    user_id = WshNetworkObject.UserName

    public_key_path = public_key_dir + CStr(user_id) + ".key"

    Open public_key_path For Output As #1

        Print #1, CStr(n)

    Close #1

End Sub

' 公開鍵e = 65535　に固定するよ

Sub generate_private_key()
    ' 秘密鍵dを計算してHomeに保存する
End Sub


' 素数を計算する
' 使った素数をLANに暗号化して登録してチェックしてはじく
Function generate_prime_number()

    Randomize

    Dim rnd_min As Double: rnd_min = 10000000
    Dim rnd_max As Double: rnd_max = 1000000000
    Dim prime_number As Double: prime_number = 0
    Dim find_prime_number As Boolean: find_prime_number = False

    prime_number = Int((rnd_max - rnd_min + 1) * Rnd + rnd_min)

    Dim i As Double
    Dim j As Double
    For i = prime_number To rnd_max
        For j = 2 To Int(Sqr(i))

            If i Mod j = 0 Then
                Exit For
            End If

            If j = Int(Sqr(i)) Then
                find_prime_number = True
            End If

        Next

        If find_prime_number Then
            prime_number = i
            Exit For
        End If

    Next

    generate_prime_number = prime_number

End Function
```

## 1 to 1 チャット
1対1でチャット出来るようにもなっていました。

```basic
Public private_final_files As Long
Public private_chat_ref_time
Public pvform_opening As Boolean


Sub private_chat_initialize()
    If pvform_opening <> True Then
        Dim fso As Object
        Set fso = CreateObject("Scripting.FileSystemObject")
        Dim f
        Dim target_path As String
        target_path = ROOT_APP_PATH & "\models\user_model\"

        For Each f In fso.GetFolder(target_path).Files
            If Not Replace(Mid(f, 37, 18), ".pasta", "") = "" Then
                Dim target_name As String
                target_name = Replace(Mid(f, 37, 18), ".pasta", "")
                target_name = "@" & target_name
                Open (f) For Input As #1
                    Line Input #1, buf
                    UserForm8.ComboBox1.AddItem target_name
                Close #1
            End If
        Next f
        MsgBox ("Initialize")
        private_chat_ref_time = Now
        For i = 0 To 10000 '自動更新
            Application.OnTime DateAdd("s", i, private_chat_ref_time), "update_form"
        Next
    Else
        pvform_opening = False
    End If
End Sub

Sub private_chat_shutdown()
    Dim subtime: subtime = DateDiff("s", private_chat_ref_time, Now)
    For i = subtime + 5 To 10000
        Application.OnTime DateAdd("s", i, private_chat_ref_time), "update_form", , False
    Next
End Sub

Sub private_chat_submit()
    Dim textbox_data As String
    textbox_data = UserForm8.TextBox1.Value

    If Not textbox_data = "" Then
        Dim message_data As String
        message_data = textbox_data

        Dim userid As String: userid = init.user_id
        Dim target_path As String
        target_path = ROOT_APP_PATH & "models\message_model\" & userid & "\" & Replace(UserForm8.ComboBox1.Value, "@", "") & "\"
        target_path = target_path & get_current_number_of_files(target_path) + 1 & ".pasta"

        Open target_path For Output As #1
            Print #1, easySecure.easy_secure(textbox_data)
            Print #1, easySecure.easy_secure(Now)
            Print #1, easySecure.easy_secure(userid)
        Close #1

        target_path = ROOT_APP_PATH & "models\message_model\" & Replace(UserForm8.ComboBox1.Value, "@", "") & "\" & userid & "\"
        target_path = target_path & get_current_number_of_files(target_path) + 1 & ".pasta"

        Open target_path For Output As #1
            Print #1, easySecure.easy_secure(textbox_data)
            Print #1, easySecure.easy_secure(Now)
            Print #1, easySecure.easy_secure(userid)
        Close #1

        UserForm8.TextBox1.Value = ""
    End If

    Call private_talk.update_form
End Sub

Sub update_form()
    pvform_opening = True
    Dim to_user As String: to_user = Replace(UserForm8.ComboBox1.Value, "@", "")
    Dim target_path As String: target_path = ROOT_APP_PATH & "models\message_model\"
    target_path = target_path & init.user_id & "\" & to_user & "\"
    Dim number_of_files As Long
    number_of_files = public_timeline.get_current_number_of_files(target_path)
    If Not Dir(target_path & private_final_files + 1 & ".pasta") = "" Then
        UserForm8.ListBox1.Clear
        Dim column_index As Long: column_index = 1
        For i = number_of_files To 1 Step -1
            column_index = 1
            Open (target_path & i & ".pasta") For Input As #1
                Do Until EOF(1)
                    Line Input #1, buf
                    If column_index = 1 Then
                        UserForm8.ListBox1.AddItem ""
                        UserForm8.ListBox1.List(UserForm8.ListBox1.ListCount - 1, 1) = easySecure.easy_r_secure(buf)
                    ElseIf column_index = 2 Then
                        UserForm8.ListBox1.List(UserForm8.ListBox1.ListCount - 1, 2) = easySecure.easy_r_secure(buf)
                    Else
                        UserForm8.ListBox1.List(UserForm8.ListBox1.ListCount - 1, 0) = easySecure.easy_r_secure(buf)
                    End If
                    column_index = column_index + 1
                Loop
            Close #1
            UserForm8.ListBox1.List(UserForm8.ListBox1.ListCount - 1, 0) = get_nickname(CStr(UserForm8.ListBox1.List(UserForm8.ListBox1.ListCount - 1, 0))) & " (@" & UserForm8.ListBox1.List(UserForm8.ListBox1.ListCount - 1, 0) & ")"
        Next
        private_final_files = number_of_files
    End If
End Sub

Sub private_chat_change_user()
    Dim target_path As String
    target_path = ROOT_APP_PATH & "models\message_model"

    If Dir(target_path & "\" & init.user_id, vbDirectory) = "" Then
        MkDir target_path & "\" & init.user_id
    End If

    If Dir(target_path & "\" & init.user_id & "\" & Replace(UserForm8.ComboBox1.Value, "@", ""), vbDirectory) = "" Then
        MkDir target_path & "\" & init.user_id & "\" & Replace(UserForm8.ComboBox1.Value, "@", "")
    End If

    If Dir(target_path & "\" & Replace(UserForm8.ComboBox1.Value, "@", ""), vbDirectory) = "" Then
        MkDir target_path & "\" & Replace(UserForm8.ComboBox1.Value, "@", "")
    End If

    If Dir(target_path & "\" & Replace(UserForm8.ComboBox1.Value, "@", "") & "\" & init.user_id, vbDirectory) = "" Then
        MkDir target_path & "\" & Replace(UserForm8.ComboBox1.Value, "@", "") & "\" & init.user_id
    End If


    private_final_files = 0
    Call private_talk.update_form
End Sub
```

## ユーザー検知
今、誰がExchatにログインしているかどうかを確かめられるようになっていました。

```basic:ユーザー検知
 Sub online()
    Dim target_path As String
    target_path = ROOT_APP_PATH & "models\user_model\online\"
    Open (target_path & init.user_id & ".pasta") For Output As #1
        Print #1, "1"
    Close #1
End Sub

Sub offline()
    Dim target_path As String
    target_path = ROOT_APP_PATH & "models\user_model\online\"
    Open (target_path & init.user_id & ".pasta") For Output As #1
        Print #1, "0"
    Close #1
End Sub

Sub search_online_user()
    Dim fso As Object
    Set fso = CreateObject("Scripting.FileSystemObject")
    Dim f
    Dim target_path As String
    target_path = ROOT_APP_PATH & "models\user_model\online"

    UserForm3.ListBox1.Clear
    For Each f In fso.GetFolder(target_path).Files
        If Not Replace(Mid(f, 44, 25), ".pasta", "") = "" Then
            Dim target_name As String
            target_name = get_nickname(Replace(Mid(f, 44, 25), ".pasta", ""))
            Open (f) For Input As #1
                Line Input #1, buf
                If buf = 1 Then
                    UserForm3.ListBox1.AddItem ""
                    UserForm3.ListBox1.List(UserForm3.ListBox1.ListCount - 1, 0) = target_name & " (@" & Replace(Mid(f, 44, 25), ".pasta", "") & ")"
                End If
            Close #1
        End If
    Next f
End Sub
```

## バックアップ
LAN内ではディレクトリの移動が激しく、よく壊れたので、バックアップが各生徒のみがアクセスできる自分のディレクトリに自動で取られ、復旧できるようになっていました。


```basic:バックアップ
Sub backup()
    If Dir("Z:\backup", vbDirectory) = "" Then
        MkDir ("Z:\backup")
    End If

    Dim target_path As String
    target_path = "Z:\backup\" & Format(Now, "yyyy-mm-dd-hh-mm-ss")

    MkDir (target_path)

    Dim PUBLIC_CHAT_PATH As String
    PUBLIC_CHAT_PATH = ROOT_APP_PATH & "models\message_model\public\"

    Dim number_of_files As Long
    number_of_files = public_timeline.get_current_number_of_files(PUBLIC_CHAT_PATH)

    Dim chat_data_array() As String
    ReDim chat_data_array(number_of_files, 4)

    Dim chat_data_index As Long
    chat_data_index = 0

    For i = 1 To number_of_files
        Open (PUBLIC_CHAT_PATH & i & ".pasta") For Input As #1
            chat_data_index = 0
            Do Until EOF(1)
                Line Input #1, buf
                chat_data_array(i - 1, chat_data_index) = buf
                chat_data_index = chat_data_index + 1
            Loop
        Close #1
    Next

    Open (target_path & "\public_chat.pasta") For Output As #1
        For i = 0 To number_of_files - 1
            Print #1, chat_data_array(i, 0)
            Print #1, chat_data_array(i, 1)
            Print #1, chat_data_array(i, 2)
            Print #1, chat_data_array(i, 3)
        Next
    Close #1

End Sub
```

## 送信取り消し
送信取り消しが実装されていました。

```basic:送信取り消し
Private Sub ListBox1_DblClick(ByVal Cancel As MSForms.ReturnBoolean)

    Dim all_messages_count As Long
    all_messages_count = public_timeline.get_current_number_of_files(ROOT_APP_PATH & "models\message_model\public\")

    Dim target_id As Long
    target_id = all_messages_count - ListBox1.ListIndex

    Dim sending_user As String

    Dim column_index As Long
    column_index = 1

    Dim raw_row1 As String
    Dim raw_row2 As String
    Dim raw_row3 As String

    Open (ROOT_APP_PATH & "models\message_model\public\" & target_id & ".pasta") For Input As #1
        Do Until EOF(1)
            Line Input #1, buf
            If column_index = 1 Then
                raw_row1 = buf
            ElseIf column_index = 2 Then
                raw_row2 = buf
            ElseIf column_index = 3 Then
                sending_user = easySecure.easy_r_secure(buf)
                raw_row3 = buf
            End If
            column_index = column_index + 1
        Loop
    Close #1


    If init.user_id = sending_user Then

        If MsgBox("送信取り消ししますか？（develop v.6.4より前のバージョンを利用するユーザーには取り消しされません）", vbOKCancel) = vbOK Then
            Open (ROOT_APP_PATH & "models\message_model\public\" & target_id & ".pasta") For Output As #1
                Print #1, raw_row1
                Print #1, raw_row2
                Print #1, raw_row3
                Print #1, "delete"
            Close #1
        End If

        UserForm4.ListBox1.Clear
        For i = number_of_files To 1 Step -1
            column_index = 1
            Open (target_path & i & ".pasta") For Input As #1
                Do Until EOF(1)
                    Line Input #1, buf
                    If column_index = 1 Then
                        UserForm4.ListBox1.AddItem ""
                        UserForm4.ListBox1.List(UserForm4.ListBox1.ListCount - 1, 1) = easySecure.easy_r_secure(buf)
                    ElseIf column_index = 2 Then
                        UserForm4.ListBox1.List(UserForm4.ListBox1.ListCount - 1, 2) = easySecure.easy_r_secure(buf)
                    ElseIf column_index = 3 Then
                        UserForm4.ListBox1.List(UserForm4.ListBox1.ListCount - 1, 0) = easySecure.easy_r_secure(buf)
                    ElseIf column_index = 4 Then
                        If buf = "delete" Then
                            UserForm4.ListBox1.List(UserForm4.ListBox1.ListCount - 1, 1) = "<送信者によってメッセージが取り消されました>"
                        End If
                    End If
                    column_index = column_index + 1
                Loop
            Close #1
            UserForm4.ListBox1.List(UserForm4.ListBox1.ListCount - 1, 0) = get_nickname(CStr(UserForm4.ListBox1.List(UserForm4.ListBox1.ListCount - 1, 0))) & " (@" & UserForm4.ListBox1.List(UserForm4.ListBox1.ListCount - 1, 0) & ")"
        Next
        final_files = number_of_files
    ElseIf init.student_number = "160004" Then
        If MsgBox("管理者権限で削除しますか？（develop v.6.4より前のバージョンを利用するユーザーには取り消しされません）", vbOKCancel) = vbOK Then
            Open (ROOT_APP_PATH & "models\message_model\public\" & target_id & ".pasta") For Output As #1
                Print #1, raw_row1
                Print #1, raw_row2
                Print #1, raw_row3
                Print #1, "admin-delete"
            Close #1
        End If

        UserForm4.ListBox1.Clear
        For i = number_of_files To 1 Step -1
            column_index = 1
            Open (target_path & i & ".pasta") For Input As #1
                Do Until EOF(1)
                    Line Input #1, buf
                    If column_index = 1 Then
                        UserForm4.ListBox1.AddItem ""
                        UserForm4.ListBox1.List(UserForm4.ListBox1.ListCount - 1, 1) = easySecure.easy_r_secure(buf)
                    ElseIf column_index = 2 Then
                        UserForm4.ListBox1.List(UserForm4.ListBox1.ListCount - 1, 2) = easySecure.easy_r_secure(buf)
                    ElseIf column_index = 3 Then
                        UserForm4.ListBox1.List(UserForm4.ListBox1.ListCount - 1, 0) = easySecure.easy_r_secure(buf)
                    ElseIf column_index = 4 Then
                        If buf = "admin-delete" Then
                            UserForm4.ListBox1.List(UserForm4.ListBox1.ListCount - 1, 1) = "<管理者によってメッセージが削除されました>"
                        End If
                    End If
                    column_index = column_index + 1
                Loop
            Close #1
            UserForm4.ListBox1.List(UserForm4.ListBox1.ListCount - 1, 0) = get_nickname(CStr(UserForm4.ListBox1.List(UserForm4.ListBox1.ListCount - 1, 0))) & " (@" & UserForm4.ListBox1.List(UserForm4.ListBox1.ListCount - 1, 0) & ")"
        Next
        final_files = number_of_files
    End If
End Sub
```

## メッセージの自動更新
他のユーザーによって送信されたメッセージは、自動更新されて表示されていたようです。

```basic:自動更新
Sub Update()
    On Error GoTo Er_line
    Dim target_path As String: target_path = ROOT_APP_PATH & "models\message_model\public\"
    Dim number_of_files As Long
    number_of_files = public_timeline.get_current_number_of_files(target_path)
    If Not Dir(target_path & final_files + 1 & ".pasta") = "" Then
        UserForm4.ListBox1.Clear
        Dim column_index As Long: column_index = 1
        For i = number_of_files To 1 Step -1
            column_index = 1
            Open (target_path & i & ".pasta") For Input As #1
                Do Until EOF(1)
                    Line Input #1, buf
                    If column_index = 1 Then
                        UserForm4.ListBox1.AddItem ""
                        UserForm4.ListBox1.List(UserForm4.ListBox1.ListCount - 1, 1) = easySecure.easy_r_secure(buf)
                    ElseIf column_index = 2 Then
                        UserForm4.ListBox1.List(UserForm4.ListBox1.ListCount - 1, 2) = easySecure.easy_r_secure(buf)
                    ElseIf column_index = 3 Then
                        UserForm4.ListBox1.List(UserForm4.ListBox1.ListCount - 1, 0) = easySecure.easy_r_secure(buf)
                    ElseIf column_index = 4 Then
                        If buf = "delete" Then
                            UserForm4.ListBox1.List(UserForm4.ListBox1.ListCount - 1, 1) = "<送信者によってメッセージが取り消されました>"
                        ElseIf buf = "admin-delete" Then
                            UserForm4.ListBox1.List(UserForm4.ListBox1.ListCount - 1, 1) = "<管理者によってメッセージが削除されました>"
                        End If
                    End If
                    column_index = column_index + 1
                Loop
            Close #1
            UserForm4.ListBox1.List(UserForm4.ListBox1.ListCount - 1, 0) = get_nickname(CStr(UserForm4.ListBox1.List(UserForm4.ListBox1.ListCount - 1, 0))) & " (@" & UserForm4.ListBox1.List(UserForm4.ListBox1.ListCount - 1, 0) & ")"
        Next
        final_files = number_of_files
    End If
    target_path = ROOT_APP_PATH & "models\information_model\"
    If Not Dir(target_path & info_number_of_files + 1 & ".pasta") = "" Then
        Dim info_data As String
        Dim line_index As Long: line_index = 1
        Dim error_data As Boolean
        Open (target_path & info_number_of_files + 1 & ".pasta") For Input As #1
            Do Until EOF(1)
                Line Input #1, buf
                If line_index = 1 Then
                    info_data = easySecure.easy_r_secure(buf)
                Else
                    error_data = CBool(buf)
                End If
                line_index = line_index + 1
            Loop
        Close #1
        If error_data Then
            MsgBox info_data, vbCritical
        Else
            MsgBox info_data, vbOKOnly
        End If
        info_number_of_files = info_number_of_files + 1
    End If
    Exit Sub
Er_line:
    Console ("エラー内容: " & Err.Description)
End Sub
```
# 終わりに
一年ぶりにコードを読んでみると、酷いものだなと思いながら、よくExcelでこんなアプリケーションを保守していたなと思います。
突然動かなくなったり（主にフォルダが動かされるのが原因だった）、機能を追加したら他の機能が死んだり、典型的な "ヤバい" 開発の沼にはまって泣きながら学校に残って作業していたことを思い出します。
当時から

```basic:リファクタリングの跡
Sub Public_New(ByRef content As String)
    If Present_(content) Then
        Const LATEST_NUM As Long = Get_Files_Count_(PUBCHAT_CONTENT_PATH) + 1

        Open PUBCHAT_CONTENT_PATH & LASTEST_NUM & PASTA For Output As #1
            Print #1, content
        Close #1

        Open PUBCHAT_TIME_PATH & LASTEST_NUM & PASTA For Output As #1
            Print #1, Now
        Close #1

        Open PUBCHAT_USERID_PATH & LASTEST_NUM & PASTA For Output As #1
            Print #1, init.user_id
        Close #1

        Open PUBCHAT_STATUS_PATH & LASTEST_NUM & PASTA For Output As #1
            Print #1, "normal"
        Close #1

        content = ""
    End If
End Sub

Sub test_submit()
    Public_New (Cells(1, 1).Value)
End Sub

Private Function Present_(ByVal data As String)
    If data = "" Then
        Present_ = False
    Else
        Present_ = True
    End If
End Function
```

リファクタリングに取り組もうとはしていたみたいです。

実際、Visual Basic for Applicationの開発環境はお世辞にも良いと言えるものではなく、かなりストレスが溜まりました。
しかし、このアプリケーションによって授業中に知見交換ができたり、タイピング練習ができたり（機能に入っていました）、なかなか楽しかったので作ってよかったなと思います。
みなさんも、プログラミング始めたての頃に作ったアプリケーションを掘り出してみると、楽しいかもしれません。
