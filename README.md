// ==UserScript==
// @name          Get Data from Steam / SteamDB / EpicDB
// @namespace     sak32009-get-data-from-steam-steamdb-epicdb
// @description   Get Data from Steam / SteamDB / EpicDB
// @author        Sak32009
// @year          2016 - 2021
// @version       4.2.3
// @license       MIT
// @homepageURL   https://github.com/Sak32009/GetDLCInfoFromSteamDB/
// @supportURL    https://github.com/Sak32009/GetDLCInfoFromSteamDB/issues/
// @updateURL     https://raw.githack.com/Sak32009/GetDLCInfoFromSteamDB/master/sak32009-get-dlc-info-from-steamdb.user.js
// @downloadURL   https://raw.githack.com/Sak32009/GetDLCInfoFromSteamDB/master/sak32009-get-dlc-info-from-steamdb.user.js
// @icon          https://rawcdn.githack.com/Sak32009/GetDLCInfoFromSteamDB/33433ac6e0910e980fa8e14a0a8c785736134c41/sak32009-get-dlc-info-from-steamdb-icon.png
// @match         *://steamdb.info/app/*
// @match         *://steamdb.info/depot/*
// @match         *://store.steampowered.com/app/*
// @match         *://sak32009.github.io/app/*
// @run-at        document-end
// @require       https://cdnjs.cloudflare.com/ajax/libs/jquery/3.6.0/jquery.min.js
// @grant         GM_xmlhttpRequest
// @grant         GM_addStyle
// @grant         unsafeWindow
// ==/UserScript==

GM_info.script.year = "2016 - 2021";
GM_info.script.homepageURL =
  "https://github.com/Sak32009/GetDLCInfoFromSteamDB/";

class SK {
  constructor(data) {
    this.formats = data;
    this.steam = {
      appID: "",
      name: "",
      header: "",
      dlcs: {},
      count: 0,
      dlcsUnknowns: {},
      countUnknowns: 0,
      appURL: "https://steamdb.info/app/",
      depotURL: "https://steamdb.info/depot/",
      steamAPI: "https://store.steampowered.com/api/appdetails?appids=",
    };
    this.localURL = "https://sak32009.github.io/app/";
  }
  run(document) {
    const self = this;
    const url = new URL(window.location.href);
    const $_GET = new URLSearchParams(url.search);
    const isSteamDBApp =
      url.hostname === "steamdb.info" && url.pathname.startsWith("/app/");
    const isSteamPoweredApp =
      url.hostname === "store.steampowered.com" &&
      url.pathname.startsWith("/app/");
    const isSteamDBDepot =
      url.hostname === "steamdb.info" &&
      url.pathname.startsWith("/depot/") &&
      $_GET.has("show_hashes");
    const isLocal =
      url.hostname === "sak32009.github.io" && url.pathname === "/app/";
    if (isLocal) {
      $("div#userscript").html(document);
      $("*[data-userscript='version']").text(GM_info.script.version);
      $("*[data-userscript='year']").text(GM_info.script.year);
      if ($_GET.has("appid")) {
        const $GET_appID = $_GET.get("appid").toString();
        const $GET_from =
          ($_GET.has("from") && $_GET.get("from").toString()) || "steamdb";
        const allowedFrom = ["steam", "steamdb", "epicdb"];
        if (
          $GET_appID.length &&
          !isNaN($GET_appID) &&
          $.inArray($GET_from, allowedFrom) !== -1
        ) {
          switch ($GET_from) {
            case "steam":
              $("div#steamOfficialSelected > h4 > span").removeClass("d-none");
              $("div#steamOfficialSelected > form > input[name='appid']").val(
                $GET_appID
              );
              self.steam.appID = $GET_appID;
              self.steam_setDLCSRequests();
              break;
            case "steamdb":
              $("div#steamDBSelected > h4 > span").removeClass("d-none");
              $("div#steamDBSelected > form > input[name='appid']").val(
                $GET_appID
              );
              self.steam.appID = $GET_appID;
              self.steamDB_setDLCSRequests();
              break;
            case "epicdb":
              $("div#epicDBSelected > h4 > span").removeClass("d-none");
              $("div#epicDBSelected > form > input[name='appid']").val(
                $GET_appID
              );
              $("div#alert").text("Its disabled for now! Sorry!");
              break;
            default:
              $("div#alert").text("Isn't a valid choice!");
          }
        } else {
          $("div#alert").text("Invalid _appID_ or _from_ data!");
        }
      } else {
        $("div#alert").text("Unknown _appID_ or _from_ data!");
      }
    } else if (isSteamDBApp || isSteamPoweredApp) {
      self.steam.appID = $("div[data-appid]").data("appid").toString();
      if (self.steam.appID.length && !isNaN(self.steam.appID)) {
        self.createInterfaceButtons();
      }
    } else if (isSteamDBDepot) {
      self.createInterfaceDepots();
    }
  }
  createInterfaceButtons() {
    const self = this;
    GM_addStyle(
      `#WIZZpeAmov a{display:block;background-color:#4b2e52;padding:8px;border:1px solid #000;border-radius:5px;text-decoration:none;color:#fff;text-align:center;font-weight:700;position:fixed;bottom:0;right:0;margin:0;margin-right:10px;z-index:999;border-bottom-left-radius:0;border-bottom-right-radius:0}#WIZZpeAmov a:hover{color:#fff;background-color:#522e47}`
    );
    $(
      `<div id="WIZZpeAmov"><a href="${self.localURL}?from=steamdb&appid=${self.steam.appID}" target="_blank">${GM_info.script.name} v${GM_info.script.version} <small>by ${GM_info.script.author} | ${GM_info.script.year}</small></a></div>`
    ).appendTo("body");
  }
  steam_setDLCSRequests() {
    const self = this;
    GM_xmlhttpRequest({
      url: `${self.steam.steamAPI + self.steam.appID}`,
      method: "GET",
      onload({ responseText }) {
        const json = JSON.parse(responseText);
        if (typeof json !== "undefined") {
          const app = json[self.steam.appID];
          if (app.success === true) {
            self.steam.name = app.data.name;
            self.steam.header = app.data.header_image;
            if (typeof app.data.dlc !== "undefined" && app.data.dlc.length) {
              $.each(app.data.dlc, (_index, appID) => {
                self.steam.dlcs[appID] = "NO NAME (For now..)";
                self.steam.count += 1;
              });
            }
            self.steam_afterDLCSRequests();
          } else {
            $("div#alert").text("Unknown error from steam!");
          }
        } else {
          $("div#alert").text("Unknown error from steam!");
        }
      },
    });
  }
  steamDB_setDLCSRequests() {
    const self = this;
    GM_xmlhttpRequest({
      url: `${self.steam.appURL + self.steam.appID}`,
      method: "GET",
      onload({ responseText }) {
        const $dom = $($.parseHTML(responseText));
        self.steam.name = $dom.find(".pagehead > h1").text().trim();
        self.steam.header = $dom
          .find("img.app-logo[itemprop='image']")
          .attr("src");
        $dom.find("#dlc.tab-pane tr.app[data-appid]").each((_index, _dom) => {
          const $dom = $(_dom);
          const appID = $dom.attr("data-appid");
          const appName = $dom.find("td:nth-of-type(2)").text().trim();
          if ($dom.find("td:nth-of-type(2)").hasClass("muted")) {
            self.steam.dlcsUnknowns[appID] = appName;
            self.steam.countUnknowns += 1;
          } else {
            self.steam.dlcs[appID] = appName;
            self.steam.count += 1;
          }
        });
        self.steam_afterDLCSRequests();
      },
    });
  }
  steam_afterDLCSRequests() {
    const self = this;
    $("*[data-app='appid']").text(self.steam.appID);
    $("*[data-app='name']").text(self.steam.name);
    $("*[data-app='count']").text(self.steam.count);
    $("*[data-app='countUnknowns']").text(self.steam.countUnknowns);
    $("*[data-app='header']").attr("src", self.steam.header);
    $("div#alert").hide();
    $("div#container").show();
    return new steam().run(self);
  }
  toBlob(name, content, extension) {
    return {
      name: `${
        name.toString().length > 0
          ? name
          : Math.random().toString(36).substring(2)
      }.${extension}`,
      blob: window.URL.createObjectURL(
        new Blob([content.replace(/\n/g, "\r\n")], {
          type: "application/octet-stream;charset=utf-8",
        })
      ),
    };
  }
  createInterfaceDepots() {
    const self = this;
    const depotID = $(`div[data-depotid]`).data("depotid");
    const depotTable = unsafeWindow
      .$("div#files .table.file-tree")
      .DataTable()
      .data();

    if (depotTable.length < 1) {
      return;
    }

    let output = `; ${GM_info.script.name} v${GM_info.script.version} by ${
      GM_info.script.author
    } | ${GM_info.script.year} | DEPOT URL: ${self.steam.depotURL + depotID}\n`;

    $.each(depotTable, (_index, _values) => {
      output += `${_values[1]} *${_values[0]}\n`;
    });

    const toBlob = self.toBlob(depotID, output, "sha1");
    $(`<h2><a href="${toBlob.blob}" download="${toBlob.name}" style="display:block;text-align:center;">Download .sha1</a></h2>
<textarea rows="20" style="width:100%;resize:none">${output}</textarea>`).insertBefore(
      "div#files > *:first-child"
    );
  }
}

class steam {
  run(main) {
    const self = this;
    self.main = main;
    self.formats = self.main.formats.steam;
    self.steam = self.main.steam;
    self.fillInterface();
    self.loadEvents();
  }
  fillInterface() {
    const self = this;
    $.each(self.formats, (_index, _values) => {
      const name = _values.name;
      $("<option>")
        .attr("value", _index)
        .text(name)
        .appendTo(`select#steam_select`);
    });
  }
  loadEvents() {
    const self = this;
    $(document).on("click", `button#steam_convert`, (e) => {
      e.preventDefault();
      const selected = $(`select#steam_select option:selected`).val();
      const withDLCSUnknowns = $("input#steam_unknowns").is(":checked");
      const data = self.formats[selected];
      const result = self.bbcode(data.file.text, withDLCSUnknowns);
      const file = self.main.toBlob(
        self.bbcode(data.file.name, false),
        result,
        data.file.ext
      );
      $(`textarea#steam_textarea`).html(result).scrollTop(0);
      $(`a#steam_download`).attr({
        href: file.blob,
        download: file.name,
      });
    });
    $(document).on("change", "input#steam_interfaces_file", ({ target }) => {
      const files = target.files;
      if (files.length > 0) {
        const file = files[0];
        const reader = new FileReader();
        const search = [
          "SteamClient",
          "SteamGameServer",
          "SteamGameServerStats",
          "SteamUser",
          "SteamFriends",
          "SteamUtils",
          "SteamMatchMaking",
          "SteamMatchMakingServers",
          "STEAMUSERSTATS_INTERFACE_VERSION",
          "STEAMAPPS_INTERFACE_VERSION",
          "SteamNetworking",
          "STEAMREMOTESTORAGE_INTERFACE_VERSION",
          "STEAMSCREENSHOTS_INTERFACE_VERSION",
          "STEAMHTTP_INTERFACE_VERSION",
          "STEAMUNIFIEDMESSAGES_INTERFACE_VERSION",
          "STEAMUGC_INTERFACE_VERSION",
          "STEAMAPPLIST_INTERFACE_VERSION",
          "STEAMMUSIC_INTERFACE_VERSION",
          "STEAMMUSICREMOTE_INTERFACE_VERSION",
          "STEAMHTMLSURFACE_INTERFACE_VERSION_",
          "STEAMINVENTORY_INTERFACE_V",
          "SteamController",
          "SteamMasterServerUpdater",
          "STEAMVIDEO_INTERFACE_V",
          "STEAMCONTROLLER_INTERFACE_VERSION",
        ];
        reader.onload = (
          () =>
          ({ target }) => {
            const content = target.result;
            let result = [];
            $.each(search, (_index, _value) => {
              const re = new RegExp(`${_value}\\d{3}`, "g");
              $.each(content.match(re), (__index, __value) => {
                result.push(__value);
              });
            });
            result = result.join("\n");
            const file = self.main.toBlob("steam_interfaces", result, "txt");
            $(`textarea#steam_interfaces_textarea`).html(result).scrollTop(0);
            $(`a#steam_interfaces_download`).attr({
              href: file.blob,
              download: file.name,
            });
          }
        )(file);
        reader.readAsText(file);
      }
    });
  }
  bbcodeDLCSReplace(str, values) {
    $.each(values, (_index, _values) => {
      str = str.replace(new RegExp(`{${_index}}`, "g"), _values);
    });
    return str;
  }
  bbcodeDLCSPrefix(index, prefix) {
    return prefix > index.length
      ? "0".repeat(prefix - index.length) + index
      : index;
  }
  bbcodeDLCS(str, indexFromZero, indexPrefix, withDLCSUnknowns) {
    const self = this;
    let result = "";
    let index = indexFromZero ? 0 : -1;
    const dlcs = withDLCSUnknowns
      ? {
          ...self.steam.dlcs,
          ...self.steam.dlcsUnknowns,
        }
      : self.steam.dlcs;
    $.each(dlcs, (_appid, _name) => {
      index += 1;
      result += self.bbcodeDLCSReplace(str, {
        dlc_id: _appid,
        dlc_name: _name,
        dlc_index: self.bbcodeDLCSPrefix(
          index.toString(),
          parseInt(indexPrefix)
        ),
      });
    });
    return result;
  }
  bbcode(str, withDLCSUnknowns) {
    const self = this;
    let data = "";
    const re = /\[(\w+)(?:=(.*))?]([^[]+)\[\/(\w+)]/g;
    while ((data = re.exec(str)) !== null) {
      const [bbcode, bbcodeOpen, bbcodeOpt, bbcodeVal, bbcodeClose] = data;
      if (bbcodeOpen === bbcodeClose) {
        const bbcodeOpts =
          typeof bbcodeOpt !== "undefined" ? bbcodeOpt.split(":") : [];
        switch (bbcodeOpen) {
          case "steam": {
            str = str.replace(bbcode, self.steam[bbcodeVal]);
            break;
          }
          case "dlcs": {
            str = str.replace(
              bbcode,
              self.bbcodeDLCS(
                bbcodeVal.replace(/\\n/g, "\n"),
                bbcodeOpts[0] === "true",
                bbcodeOpts[1] || 0,
                withDLCSUnknowns
              )
            );
            break;
          }
        }
      }
    }
    return str;
  }
}

const a = new SK({
  steam: [
    {
      name: "CreamAPI v4.5.0.0",
      file: {
        name: "cream_api",
        ext: "ini",
        text: `[steam]
; Application ID (http://store.steampowered.com/app/%appid%/)
appid = [steam]appID[/steam]
; Current game language.
; Uncomment this option to turn it on.
; Default is "english".
;language = german
; Enable/disable automatic DLC unlock. Default option is set to "false".
; Keep in mind that this option  WON'T work properly if the "[dlc]" section is NOT empty
unlockall = false
; Original Valve's steam_api.dll.
; Default is "steam_api_o.dll".
orgapi = steam_api_o.dll
; Original Valve's steam_api64.dll.
; Default is "steam_api64_o.dll".
orgapi64 = steam_api64_o.dll
; Enable/disable extra protection bypasser.
; Default is "false".
extraprotection = false
; The game will think that you're offline (supported by some games).
; Default is "false".
forceoffline = false
; Some games are checking for the low violence presence.
; Default is "false".
;lowviolence = true
; Purchase timestamp for the DLC (http://www.onlineconversion.com/unix_time.htm).
; Default is "0" (1970/01/01).
;purchasetimestamp = 0

[steam_misc]
; Disables the internal SteamUser interface handler.
; Does have an effect on the games that are using the license check for the DLC/application.
; Default is "false".
disableuserinterface = false

[dlc]
; DLC handling.
; Format: <dlc_id> = <dlc_description>
; e.g. : 247295 = Saints Row IV - GAT V Pack
; If the DLC is not specified in this section
; then it won't be unlocked
[dlcs]{dlc_id} = {dlc_name}\n[/dlcs]`,
      },
    },
    {
      name: "CreamAPI v3.4.1.0",
      file: {
        name: "cream_api",
        ext: "ini",
        text: `[steam]
; Application ID (http://store.steampowered.com/app/%appid%/)
appid = [steam]appID[/steam]
; Current game language.
; Uncomment this option to turn it on.
; Default is "english".
;language = german
; Enable/disable automatic DLC unlock. Default option is set to "false".
; Keep in mind that this option is highly experimental and won't
; work if the game wants to call each DLC by index.
unlockall = false
; Original Valve's steam_api.dll.
; Default is "steam_api_o.dll".
orgapi = steam_api_o.dll
; Original Valve's steam_api64.dll.
; Default is "steam_api64_o.dll".
orgapi64 = steam_api64_o.dll
; Enable/disable extra protection bypasser.
; Default is "false".
extraprotection = false
; The game will think that you're offline (supported by some games).
; Default is "false".
forceoffline = false
; Some games are checking for the low violence presence.
; Default is "false".
;lowviolence = true
; Installation path for the game.
; Note, that you can use ..\\ to set the parent directory (from where executable file is located).
; Maximum number of parent directories: 5 (..\\..\\..\\..\\..\\)
; Default is the path to current working directory.
;installdir = ..\\
; Use DLC id as the appended installation directory.
; e.g. <install_directory>\\480
; Default is "true".
;dlcasinstalldir = false
; Purchase timestamp for the DLC (http://www.onlineconversion.com/unix_time.htm).
; Default is "0" (1970/01/01).
;purchasetimestamp = 0
; Turn on the wrapper mode.
; Default is "false".
wrappermode = false

[steam_misc]
; Disables the internal SteamUser interface handler.
; Does have an effect on the games that are using the license check for the DLC/application.
; Default is "false".
disableuserinterface = false
; Disables the internal SteamUtils interface handler.
; Does have an effect on the games that are checking for the actual AppId (only matters when "wrappermode" is set to "true").
; Default is "false".
disableutilsinterface = false
; Disable the internal reserve hook of the "Steam_RegisterInterfaceFuncs" function.
; Default is "false".
disableregisterinterfacefuncs = false
; Unlock/Lock Steam parental restrictions.
; Default is "true".
;unlockparentalrestrictions = false
; SteamId64 to override. Note that this action could be risky !
; This option can only work if "disableuserinterface = false".
;steamid = 0
; Bypass VAC signature check. Note that this action could be risky !
; Default is "false".
;signaturebypass = true

[steam_wrapper]
; Application ID to override (used when the wrapper mode is on)
newappid = 0
; Use the internal storage system.
; Default is "false".
wrapperremotestorage = false
; Use the internal stats/achievements system.
; Default is "false".
wrapperuserstats = false
; Use the internal workshop (UGC) system.
; Default is "false".
wrapperugc = false
; Store the data in the current directory (incl. stats)
; By default the data is stored at: %appdata%/CreamAPI/%appid%/
; Default is "false".
saveindirectory = false
; Force the usage of a full save path instead of the relative one.
; Default is "false".
forcefullsavepath = false
; Disable internal callbacks system.
; Default is "false".
;disablecallbacks = true
; Disable/Enable a StoreStats callback. Takes effect only if "wrapperuserstats" is set to "true".
; Default is "true".
;storestatscallback = false
; Fixed achievements count.
; Some games can only work if this option is configured properly (e.g. Wolfenstein II).
; Default is "0".
achievementscount = 0

[dlc]
; DLC handling.
; Format: <dlc_id> = <dlc_description>
; e.g. : 247295 = Saints Row IV - GAT V Pack
; If the DLC is not specified in this section
; then it won't be unlocked
[dlcs]{dlc_id} = {dlc_name}\n[/dlcs]
[dlc_installdirs]
; Installation path for the specific DLC (dependent from "installdir" option).
; This section works only if "dlcasinstalldir" option is set to "false".
; Format: <dlc_id> = <install_dir>
; e.g. : 556760 = DLCRoot0

[steam_ugc]
; Subscribed workshop items.
; This section works only if "wrappermode" and "wrapperugc" options are set to "true".
; Format: <dlc_id> = <true/false>
; e.g. : 812713531 = true
; Please refer to __README_WORKSHOP_EN__.txt for more details.`,
      },
    },
    {
      name: "GreenLuma 2020 [BATCH MODE]",
      file: {
        name: "[steamdb]appID[/steamdb]_GreenLuma",
        ext: "bat",
        text: `@ECHO OFF
:: WINDOWS WORKING DIR BUG WORKAROUND
CD /D "%~dp0"
:: CHECK APPLIST DIR
IF EXIST .\\AppList RMDIR /S /Q .\\AppList
:: CREATE APPLIST DIR
MKDIR .\\AppList
:: CREATE DLCS FILES FOR __[steam]name[/steam]__
ECHO [steam]appID[/steam]> .\\AppList\\0.txt
[dlcs=true]:: {dlc_name}\nECHO {dlc_id}> .\\AppList\\{dlc_index}.txt\n[/dlcs]
:: START GREENLUMA 2020
IF EXIST .\\DLLInjector.exe GOTO :Q
GOTO :EXIT
:Q
SET /P c=Do you want to start GreenLuma 2020 [Y/N]?
IF /I "%c%" EQU "Y" GOTO :START
IF /I "%c%" EQU "N" GOTO :EXIT
GOTO :Q
:START
CLS
ECHO Launching Greenluma 2020 - APPID [steam]appID[/steam] - APPNAME [steam]name[/steam]
TASKKILL /F /IM steam.exe
TIMEOUT /T 2
DLLInjector.exe -DisablePreferSystem32Images
:EXIT
EXIT`,
      },
    },
    {
      name: "LUMAEMU (ONLY DLCS LIST)",
      file: {
        name: "",
        ext: "ini",
        text: "[dlcs]; {dlc_name}\nDLC_{dlc_id} = 1\n[/dlcs]",
      },
    },
    {
      name: "CODEX (DLC00000 = DLCName)",
      file: {
        name: "",
        ext: "ini",
        text: "[dlcs=false:5]DLC{dlc_index} = {dlc_id}\nDLCName{dlc_index} = {dlc_name}\n[/dlcs]",
      },
    },
    {
      name: "3DMGAME (ONLY DLCS LIST)",
      file: {
        name: "",
        ext: "ini",
        text: "[dlcs=true:3]; {dlc_name}\nDLC{dlc_index} = {dlc_id}\n[/dlcs]",
      },
    },
    {
      name: "SKIDROW (ONLY DLCS LIST)",
      file: {
        name: "",
        ext: "ini",
        text: "[dlcs]; {dlc_name}\n{dlc_id}\n[/dlcs]",
      },
    },
    {
      name: "APPID = APPIDNAME",
      file: {
        name: "",
        ext: "ini",
        text: "[dlcs]{dlc_id} = {dlc_name}\n[/dlcs]",
      },
    },
    {
      name: "APPIDNAME",
      file: {
        name: "",
        ext: "ini",
        text: "[dlcs]{dlc_name}\n[/dlcs]",
      },
    },
  ],
});
a.run(`
<div class="row text-center">
    <div class="col-sm-4">
        <div id="steamOfficialSelected" class="m-3">
            <h4 class="text-danger pb-2">
                Steam Official appID<span class="d-none">&nbsp;<i class="fas fa-arrow-circle-down"></i></span>
            </h4>
            <form action="" method="get">
                <input type="hidden" name="from" value="steam" />
                <input type="text" class="form-control" name="appid" placeholder="Example: 550" />
            </form>
        </div>
    </div>
    <div class="col-sm-4">
        <div id="steamDBSelected" class="m-3">
            <h4 class="text-danger pb-2">
                SteamDB appID<span class="d-none">&nbsp;<i class="fas fa-arrow-circle-down"></i></span>
            </h4>
            <form action="" method="get">
                <input type="hidden" name="from" value="steamdb" />
                <input type="text" class="form-control" name="appid" placeholder="Example: 550" />
            </form>
        </div>
    </div>
    <div class="col-sm-4">
        <div id="epicDBSelected" class="m-3">
            <h4 class="text-danger pb-2">
                EpicDB appID<span class="d-none">&nbsp;<i class="fas fa-arrow-circle-down"></i></span>
            </h4>
            <form action="" method="get">
                <input type="hidden" name="from" value="epicdb" />
                <span tabindex="0" data-bs-toggle="tooltip" title="It's disabled for now! Sorry!">
                    <input type="text" class="form-control" name="appid" placeholder="Example: ?" disabled />
                </span>
            </form>
        </div>
    </div>
</div>
<div id="alert" class="alert alert-danger text-center">If it takes too long, check the console log of browser and open issue on <a href="https://github.com/Sak32009/GetDLCInfoFromSteamDB/issues" target="_blank">GitHub</a>.</div>
<div id="container" style="display: none;">
    <div class="row">
        <div class="col-sm-8">
            <div class="card text-white bg-custom-dark">
                <table class="table table-dark table-bordered table-hover mb-0">
                    <thead>
                        <tr>
                            <td colspan="2" class="text-center">
                                <i class="fas fa-info-circle"></i>
                                <span>Info</span>
                            </td>
                        </tr>
                    </thead>
                    <tbody>
                        <tr>
                            <td>
                                <i class="fas fa-fingerprint"></i>
                                <span>App ID:</span>
                            </td>
                            <td data-app="appid">?</td>
                        </tr>
                        <tr>
                            <td>
                                <i class="fas fa-signature"></i>
                                <span>App Name:</span>
                            </td>
                            <td data-app="name">?</td>
                        </tr>
                        <tr>
                            <td>
                                <i class="fas fa-signature"></i>
                                <span>Total DLCs without Unknowns:</span>
                            </td>
                            <td data-app="count">?</td>
                        </tr>
                        <tr>
                            <td>
                                <i class="fas fa-signature"></i>
                                <span>Total DLCs Unknowns:</span>
                            </td>
                            <td data-app="countUnknowns">?</td>
                        </tr>
                    </tbody>
                </table>
            </div>
        </div>
        <div class="col-sm-4">
            <img src="https://via.placeholder.com/460x215?text=App%20Header" data-app="header" class="img-fluid rounded" alt="Header" />
        </div>
    </div>
    <div class="card mt-2 text-white bg-custom-dark">
        <div class="card-header">
            <ul id="steam_tabs" class="nav nav-tabs card-header-tabs justify-content-center">
                <li class="nav-item">
                    <button type="button" class="nav-link text-white active" data-bs-toggle="tab" data-bs-target="#steam_converter">
                        <i class="fas fa-sync-alt"></i>
                        <span>Converter</span>
                    </button>
                </li>
                <li class="nav-item">
                    <span data-bs-toggle="tooltip" title="It's disabled for now! Sorry!">
                        <button type="button" class="nav-link text-white" disabled>
                            <i class="fas fa-file-import"></i>
                            <span>Drag'n'Drop (Autocomplete)</span>
                        </button>
                    </span>
                </li>
                <li class="nav-item">
                    <button type="button" class="nav-link text-white" data-bs-toggle="tab" data-bs-target="#steam_interfaces">
                        <i class="fab fa-steam"></i>
                        <span>Interfaces</span>
                    </button>
                </li>
                <li class="nav-item">
                    <span data-bs-toggle="tooltip" title="It's disabled for now! Sorry!">
                        <button type="button" class="nav-link text-white" disabled>
                            <i class="fab fa-steam"></i>
                            <span>Depots</span>
                        </button>
                    </span>
                </li>
                <li class="nav-item">
                    <span data-bs-toggle="tooltip" title="It's disabled for now! Sorry!">
                        <button type="button" class="nav-link text-white" disabled>
                            <i class="fab fa-steam"></i>
                            <span>Items</span>
                        </button>
                    </span>
                </li>
                <li class="nav-item">
                    <span data-bs-toggle="tooltip" title="It's disabled for now! Sorry!">
                        <button type="button" class="nav-link text-white" disabled>
                            <i class="fab fa-steam"></i>
                            <span>Achievements</span>
                        </button>
                    </span>
                </li>
                <li class="nav-item">
                    <span data-bs-toggle="tooltip" title="It's disabled for now! Sorry!">
                        <button type="button" class="nav-link text-white" disabled>
                            <i class="fas fa-cogs"></i>
                            <span>Configurator</span>
                        </button>
                    </span>
                </li>
            </ul>
        </div>
        <div class="card-body">
            <div class="tab-content">
                <div class="tab-pane fade show active" id="steam_converter">
                    <div class="input-group">
                        <select id="steam_select" class="form-select text-white bg-custom-dark1"></select>
                        <button id="steam_convert" class="btn btn-dark" type="button">
                            <i class="fas fa-sync-alt"></i>
                            <span>Convert</span>
                        </button>
                        <label class="btn btn-dark" for="steam_unknowns"><input class="form-check-input" type="checkbox" id="steam_unknowns" /> With DLCS Unknowns</label>
                        <a id="steam_download" href="javascript:;" class="btn btn-dark">
                            <i class="fas fa-file-download"></i>
                            <span>Download as file</span>
                        </a>
                    </div>
                    <textarea id="steam_textarea" class="form-control text-white bg-custom-dark1 mt-2" rows="20" placeholder="Select an option and click 'Convert'" style="resize: none;"></textarea>
                </div>
                <div class="tab-pane fade" id="steam_interfaces">
                    <div class="input-group">
                        <input id="steam_interfaces_file" class="form-control" type="file" accept=".dll" />
                        <a id="steam_interfaces_download" href="javascript:;" class="btn btn-dark">
                            <i class="fas fa-file-download"></i>
                            <span>Download as file</span>
                        </a>
                    </div>
                    <textarea id="steam_interfaces_textarea" class="form-control text-white bg-custom-dark1 mt-2" rows="20" placeholder="Select from the input steam_api(64).dll" style="resize: none;"></textarea>
                </div>
            </div>
        </div>
    </div>
</div>`);

