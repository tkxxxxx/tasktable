# tasktable
# coding:utf-8
from django.conf import settings
from taskCalendar.models import *
from django.db import connection

import os
import collections
import datetime

# 未集計
Incomplete = 0
# Infoに指定された処理
Info = (1, 2, 4, 8)
# 確定済み
Fixed = 16
# 欠測
Missed = 32
# 測定局が対象期間外
NotAvailable = 64

# 測定局タスク表示用デフォルト文字
DefaultTaskChar = "■"

# 測定局タスク表示用デフォルトカラー
DefaultTaskColor = "White"

# 測定局情報の取得・測定局リストへの初期セット
def getStationInfo(icao):
    with connection.cursor() as cur:
        cur.execute(getStationInfoSql(), (icao,))

        stationInfo = []
        for row in cur:
            map = {}
            map['icao'] = row[0]
            map['stationId'] = row[2]
            map['stationName'] = row[3]
            map['selected'] = row[4]
            stationInfo.append(map)

    return stationInfo

def getStationInfoSql():
    sql = 'SELECT'
    sql += ' AL.ICAO AS ICAO,'
    sql += ' AL.AirportName AS AirportName,'
    sql += '  SL.StationID AS StationID,'
    sql += '  AD.ShortName AS StationName,'
    sql += '  SL.Selected AS Selected '
    sql += 'FROM'
    sql += '  ('
    sql += '    AirportList AL'
    sql += '    INNER JOIN StationList SL ON AL.ICAO = SL.ICAO'
    sql += '  )'
    sql += '  INNER JOIN AddressSpan AD ON (SL.StationID = AD.StationID)'
    sql += '  AND (SL.ICAO = AD.ICAO) '
    sql += 'WHERE'
    sql += ' AL.ICAO IN (%s)'
    sql += '  AND AD.StartDate = ('
    sql += '    SELECT'
    sql += '      MAX(StartDate)'
    sql += '    FROM'
    sql += '      AddressSpan'
    sql += '    WHERE'
    sql += '      ICAO = SL.ICAO'
    sql += '      AND StationID = SL.StationID'
    sql += '  )'
    sql += '  AND SL.Selected <= 0 '
    sql += 'ORDER BY'
    sql += '  SL.SortKey'
    return sql

# iniファイルを読み取り
def loadIniFile():

    # ファイルパス
    FILE_DIR = os.path.join(settings.BASE_DIR, 'bin')
    file = 'TaskCalendar.ini'
    path = os.path.join(FILE_DIR, file)

    # ファイルを読み取り
    with open(path) as file_object:
        lines = file_object.readlines()

    # 戻り値
    resultMap = collections.OrderedDict()

    # 明細追加フラグ(True:追加, False:追加しない)
    detailFlg = True

    for line in lines:
        # 改行コードを除く
        line = line.strip()

        # 空行とコメントアウト行を除く
        if line == '' or line.startswith("#"):
            continue

        if line.startswith("[") and line.endswith("]"):
            map = {}
            # "[]"を除く
            key = line.replace("[", "").replace("]", "")

            for k in resultMap:
                if k == key:
                    # keyが存在する場合、追加しない
                    detailFlg = False
                    break
                else:
                    detailFlg = True

            if detailFlg:
                resultMap[key] = map
            continue

        if detailFlg:
            # 文字列の分割
            tempList = line.split("=")
            # ""を除く
            subVal = tempList[1].replace("\"", "")
        
            if tempList[0].endswith("Color"):
                # カラーの場合
                subVal = "#" + subVal

            # マップを作る
            map[tempList[0]] = subVal

    return resultMap

# 凡例表示
def setRemarks(fileMap):

    # 戻り値
    remarksList = []

    for key in fileMap:
        if key.startswith("Info"):
            # 表示する英文字を作成する。
            fileMap[key]["Character"] = chr(97 + int(key[-1]) + 1).title()
            remarksList.append(fileMap[key])

    return remarksList

# ［各局の進捗状況］リストを更新する
def setdgvStationTaskList(tDate, stationInfo, fileMap):
    for stnInfo in stationInfo:
        if stnInfo["selected"] == -1:

            # 指定日の指定測定局のタスクステータスを返す
            nStationStatus = stnInfo[stationColumn(stnInfo["icao"], stnInfo["stationId"])]

            # タスクステータスから測定局タスク表示用文字を返す
            stnInfo["char"] = getStationTaskChars(nStationStatus, fileMap)

            # タスクステータスから背景色を返す
            stnInfo["backColor"] = getBackColor(nStationStatus, fileMap)

''' <summary>
    データテーブルに指定日のタスク情報をセットし、最も進捗の遅いタスクステータスを返す
    </summary>
    <param name="tDate">日付</param>
    <param name="xStationInfo"></param>
    <returns></returns>
'''
def fillTaskTable(tDate, xStationInfo, fileMap):
    
    nRet = NotAvailable
    iSelected = 0

    try:

        for xStnInfo in xStationInfo:
            # 各測定局のタスクステータスを取得
            if xStnInfo["selected"] == -1:
                iSelected += 1
                # 進捗状況を取得する。
                nStatus = stationProgress(xStnInfo["icao"], xStnInfo["stationId"], tDate, fileMap)
                xStnInfo[stationColumn(xStnInfo["icao"], xStnInfo["stationId"])] = nStatus

                if nRet > nStatus:
                    nRet = nStatus

        if iSelected == 0 or nRet == NotAvailable:
            # 全局欠測の場合はカレンダー背景色は欠測背景色
            nRet = Incomplete

    except Exception as e:
        print("タスクテーブルの生成に失敗しました。%s" % (str(e)))
    finally:
        return nRet

''' <summary>
    進捗状況を調べる
    </summary>
    <param name="sICAO">ICAOコード</param>
    <param name="sStationID">チェック測定局</param>
    < param name="tDate">チェック日付< / param > 
    <param name="fileMap">iniファイル</param>
    < returns > 
     0：未集計
     1,2,4,8：Infoに指定された処理
     16：確定済み
     32：欠測
     64：測定局が対象期間外
''' 
def stationProgress(sICAO, sStationID, tDate, fileMap):

    nRet = Incomplete

    try:

        # 測定局測定期間チェック
        if isAvailable(sICAO, sStationID, tDate):
            # 欠測チェック
            if not isMiss(sICAO, sStationID, tDate):
                # TaskInfoStチェック
                nRet = getTaskStatus(sICAO, sStationID, tDate, fileMap)
            else:
                # 欠測
                nRet = Missed
        else:
            # 対象期間外
            nRet = NotAvailable

    except Exception as e:
        print("進捗状況確認処理中にエラーが発生しました。%s" % (str(e)))
    finally:
        return nRet

''' <summary>
    測定局が指定日において有効か判定する
    </summary>
    <param name="sICAO">空港CD</param>
    <param name="sStationID">測定局ID</param>
    <param name="tDate">日付</param>
    <returns>True：有効/False：無効</returns>
'''
def isAvailable(sICAO, sStationID, tDate):

    bRet = False
    
    try:

        # 翌日を取得する。
        nextDay = tDate + datetime.timedelta(1)

        rows = Stationspan.objects.filter(icao = sICAO, stationid = sStationID, startdate__lt = nextDay, enddate__gte = tDate)

        if len(rows) > 0:
            bRet = True

    except Exception as e:
        print("測定局の期間有効判定処理中にエラーが発生しました。%s" % (str(e)))
    finally:
        return bRet

    

''' <summary>
    指定日において測定局が欠測定義されているかを判定する
    </summary>
    <param name="sICAO">空港CD（ICAO）</param>
    <param name="sStationID">測定局ID</param>
    <param name="tDate">日付</param>
    <returns>True：欠測定義あり/False：欠測定義なし</returns>
'''
def isMiss(sICAO, sStationID, tDate):

    bRet = False
    
    try:

        # 翌日を取得する。
        nextDay = tDate + datetime.timedelta(1)

        rows = Missspan.objects.filter(icao = sICAO, stationid = sStationID, starttime__lt = nextDay, endtime__gte = tDate)
    
        if len(rows) > 0:
            # 欠測判定基準は、指定測定局・指定日のレコードが存在するか
            bRet = True

    except Exception as e:
        print("測定局の欠測判定処理中にエラーが発生しました。%s" % (str(e)))
    finally:
        return bRet

    
''' <summary>
    ■ タスク状況の取得
    </summary>
    <param name="clsDB">データベース接続オブジェクト</param>
    <param name="sICAO">空港CD（ICAO形式）</param>
    <param name="sStationID">測定局ID</param>
    <param name="tDate">対象日</param>
    <param name="fileMap">iniファイル</param>
    <returns></returns>
'''
def getTaskStatus(sICAO, sStationID, tDate, fileMap):

    nRet = Incomplete
    
    try:
        
        rows = Taskinfost.objects.filter(icao = sICAO, stationid = sStationID, measdate = tDate)

        if len(rows) > 0:
            iIdx = 0
            for key in fileMap:
                if key.startswith("Info"):
                    sColumns = fileMap[key]["Flag"].split(",")

                    # 検索条件
                    kwargs = {}
                    #kwargs['icao'] = sICAO
                    #kwargs['stationid'] = sStationID
                    #kwargs['measdate'] = tDate

                    for sCol in sColumns:
                        sCol = sCol.lower()
                        kwargs[sCol] = -1

                    #drRow = Taskinfost.objects.filter(**kwargs)
                    drRow = rows.filter(**kwargs)

                    if len(drRow) > 0:
                        nRet = Info[iIdx]

                    iIdx += 1

            if rows.first().fixed_flag == -1:
                nRet = nRet | Fixed
        else:
            nRet = Incomplete

    except Exception as e:
        print("%sの測定局タスクの取得に失敗しました。%s" % (str(tDate), str(e)))
    finally:
        return nRet

''' <summary>
    タスクステータスから測定局タスク表示用文字を返す
    </summary>
    <param name="nProgress">タスクステータス番号</param>
    <returns>測定局タスク表示用文字</returns>
'''
def getStationTaskChars(nProgress, fileMap):

    nTemp = nProgress & (~ Fixed)

    # 測定局タスク表示用デフォルト文字
    sRet = DefaultTaskChar

    try:
        if nTemp == Incomplete:
            pass
        elif nTemp == Missed:
            sRet = fileMap["Master"]["MissChar"]
        elif nTemp == NotAvailable:
            pass
        else:
            for iIdx in range(len(Info)):
                if nTemp == Info[iIdx]:
                    sRet = fileMap["Info" + str(iIdx + 1)].get("Char", DefaultTaskChar)

    except Exception as e:
        print("タスクステータスから背景色への変換に失敗しました。%s" % (str(e)))
    finally:
        return sRet
'''
タスクステータスから背景色を返す
'''
def getBackColor(nProgress, fileMap):

    nTemp = nProgress & (~ Fixed)

    # 測定局タスク表示用デフォルトカラー
    cRet = DefaultTaskColor

    try:
        if nTemp == Incomplete:
            cRet = "White"
        elif nTemp == Missed:
            cRet = fileMap["Master"]["MissBackColor"]
        elif nTemp == NotAvailable:
            cRet = "Black"
        else:
            for iIdx in range(len(Info)):
                if nTemp == Info[iIdx]:
                    cRet = fileMap["Info" + str(iIdx + 1)].get("BackColor", DefaultTaskColor)

    except Exception as e:
        print("タスクステータスから背景色への変換に失敗しました。%s" % (str(e)))
    finally:
        return cRet

'''
タスクステータスからテキスト色を返す
'''
def getForeColor(nProgress, fileMap):

    cRet = "Black"

    try:
        if Fixed == (nProgress & Fixed):
            cRet = fileMap["Master"]["FixedForeColor"]
        else:
            for iIdx in range(len(Info)):
                if Info[iIdx] == (nProgress & Info[iIdx]):
                    cRet = fileMap["Info" + str(iIdx + 1)].get("ForeColor", "Black")

    except Exception as e:
        print("タスクステータスからテキスト色への変換に失敗しました。%s" % (str(e)))
    finally:
        return cRet

# プロパティ
def stationColumn(sICAO, sStationID):
    return sICAO.strip() + "_" + sStationID.strip()


#############################TaskCalendar.ini#####################################
[Database]
dbServer="localhost"
dbUser=""
dbPassword=""
Database="titandb_osk"

#Choice="PostgreSQL"
#Connector="%Program Files%\PostgreSQL\Npgsql\ms.net4.5\Npgsql.dll"

Choice="MSA"
dbPath="D:\DLReport\Config\master.accdb"

[Master]
ICAO="RJBB"
MissBackColor="FF0000"
MissChar="■"
FixedForeColor="FF0000"

[Info1]
Flag="Noise_Flag,Environ_Flag,LAE_Flag"
BackColor="E0FFFF"
ForeColor="000080"
Char="▲"
Comment="データ登録完了。航空機騒音判定処理が完了していません。"

[Info2]
Flag="SBM_Flag"
BackColor="87CEEB"
ForeColor="000080"
Char="■"
Comment="航空機騒音判定が完了。運航実績との照合が完了していません。"

[Info3]
Flag="Adjust_Flag"
BackColor="6495ED"
ForeColor="000080"

Comment="運航実績との照合が完了。"

##[Info4]
##Flag="OSFit_Flag"
##BackColor="6495ED"
##ForeColor="000080"
##Comment="運航実績との照合が完了。"

#[Master]
#ICAO=RJCC
#MissBackColor="FF0000"
#MissChar="■"
#FixedForeColor="FF0000"

#[Info1]
#Flag="Noise_Flag,Environ_Flag,LAE_Flag"
#BackColor="E0FFFF"
#ForeColor="000080"
#Char="■"
#Comment="データ登録完了。航空交通情報（航跡）との照合が完了していません。"

#[Info2]
#Flag="ONFit_Flag"
#BackColor="87CEEB"
#ForeColor="000080"
#Char="■"
#Comment="航跡照合が完了。軍民判定、局間照合が完了していません。"

#[Info3]
#Flag="Military_Flag"
#BackColor="ff8000"
#ForeColor="000080"
#Comment="軍民判定が完了。(識別局のみ)"

#[Info4]
#Flag="SurrSta_Flag"
#BackColor="00ff00"
#ForeColor="000080"
#Comment="局間照合が完了。(騒音局のみ)"
