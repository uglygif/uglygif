

(function () {
    'use strict';

    async function LeymansGuzStreamSniper(target, placeid, debug) {

        document.getElementById("joinGameButton").style.display = "none"
        document.getElementById("display").style.display = "none"
        document.getElementById("progress").style.display = "none"
        document.getElementById("image").style.display = "none"

        var id, photo
        var cursor = ""
        var tokens = []
        var batchdone = 0
        var datastorage = {}
        var gamesfinished = false
        var starttime = Date.now()
        var found = false
        var playercount = 0
        var servercount = 0
        var reqcount = 0

        function resizeTextarea() {
            const textarea = document.getElementById("display")
            let content = textarea.value;
            let lines = content.split('\n').length;
            let height = lines * 20;
            if (lines === 1) {
                height = 36
            }
            textarea.style.height = height + 'px';
        }

        function wait(ms) {
            return new Promise(resolve => setTimeout(resolve, ms));
        }

        function waitForPhoto() {
            return new Promise(resolve => {
                const checkPhoto = () => {
                    if (photo !== undefined) {
                        resolve();
                    } else {
                        setTimeout(checkPhoto, 1);
                    }
                };
                checkPhoto();
            });
        }

        async function getid(name) {
            const data = await fetch("https://www.roblox.com/users/profile?username=" + name)
            if (data.ok) {
                id = data.url.match(/\/users\/(\d+)\/profile/)[1]
            } else {
                document.getElementById("display").style.display = ""
                document.getElementById("display").value = 'User does not exist'
                resizeTextarea()
                document.getElementById("snipeButton").style.display = ""
                document.getElementById("progress").style.display = "none"
                throw new Error("User does not exist")
            }
            const data2 = await fetch("https://thumbnails.roblox.com/v1/users/avatar-headshot?userIds=" + id + "&format=Png&size=150x150&isCircular=false")
            if (data2.ok) {
                photo = await data2.text()
                photo = JSON.parse(photo)
                photo = photo.data[0].imageUrl
                document.getElementById("image").style.display = ""
                document.getElementById("image").src = photo
                if (photo[0] !== "h") {
                    document.getElementById("display").style.display = ""
                    document.getElementById("display").value = "Moderated image found for " + target
                    resizeTextarea()
                    document.getElementById("snipeButton").style.display = ""
                    document.getElementById("progress").style.display = "none"
                    throw new Error("Moderated image")
                }
            }
        }

        async function fetchtoken(id) {
            while (cursor !== null) {
                if (found) {
                    break
                }
                const data = await fetch("https://games.roblox.com/v1/games/" + id + "/servers/0?excludeFullGames=false&limit=100&cursor=" + cursor)
                var response = await data.text()
                response = JSON.parse(response)

                if (response.data === undefined) {
                    continue
                }

                cursor = response.nextPageCursor
                var playertokendata = response.data
                servercount = servercount + playertokendata.length
                reqcount++

                for (var i = 0; i < playertokendata.length; i++) {
                    for (const [key, value] of Object.entries(playertokendata[i].playerTokens)) {
                        playercount++
                        tokens.push([id, playertokendata[i].id, value, playertokendata[i].maxPlayers, playertokendata[i].playing, playertokendata[i].ping]);
                    }
                }

                (async () => {
                    await waitForPhoto()
                    if ((!gamesfinished) && (photo[0] === "h")) {
                        document.getElementById("progress").style.display = ""
                    }
                    document.getElementById("progress").textContent = "Scanning " + playercount + " players"
                })()
            }
            gamesfinished = true
        }

        async function fetchphoto() {
            while ((!gamesfinished) || (tokens.length > 0)) {
                await wait(0)
                if (tokens.length === 0) {
                    continue
                }

                var requiredlength
                if (tokens.length < 100) {
                    requiredlength = tokens.length
                } else {
                    requiredlength = 100
                }

                const tab = []
                for (var i = 0; i < requiredlength; i++) {
                    datastorage[tokens[i][2]] = tokens[i]
                    tab.push(tokens[i][2])
                }
                tokens.splice(0, requiredlength)

                var payload = []
                for (var a = 0; a < tab.length; a++) {
                    payload.push({
                        requestId: "0:" + tab[a] + ":AvatarHeadshot:150x150:png:regular",
                        type: "AvatarHeadShot",
                        targetId: 0,
                        token: tab[a],
                        format: "png",
                        size: "150x150"
                    })
                }

                (async () => {
                    const lengthofdata = requiredlength
                    const data = await fetch("https://thumbnails.roblox.com/v1/batch", {
                        method: 'POST',
                        headers: {
                            'Content-Type': 'application/json'
                        },
                        body: JSON.stringify(payload)
                    })
                    var response = await data.text()
                    response = JSON.parse(response).data

                    for (var i = 0; i < response.length; i++) {
                        await waitForPhoto();

                        if ((response[i].imageUrl === photo) && (!found) && (photo[0] === "h")) {
                            found = true
                            const sniped = datastorage[response[i].requestId.substring(2, 34)]
                            document.getElementById("snipeButton").style.display = ""
                            document.getElementById("joinGameButton").style.display = ""
                            document.getElementById("progress").style.display = "none"
                            document.getElementById("joinGameButton").onclick = () => {
                                window.Roblox.GameLauncher.joinGameInstance(
                                    sniped[0],
                                    sniped[1]
                                );
                            };

                            if (debug) {
                                document.getElementById("display").style.display = ""
                                const printtext = "Debug info\nTarget: " + target + "\nUser ID: " + id + "\nPlace ID: " + sniped[0] + "\nJob ID: " + sniped[1] + "\nPlayer Token: " + sniped[2] + "\nPlaying: " + sniped[4] + "/" + sniped[3] + "\nPing: " + sniped[5] + "\nPhoto (Thumb): " + photo + "\nPhoto (Batch): " + response[i].imageUrl + "\nScanned Servers: " + servercount + "\nScanned Players: " + playercount + "\nRequest(s) Sent: " + reqcount + "\nTime Elapsed: " + (Date.now() - starttime) / 1000 + " seconds\nSpeed: " + Math.floor((playercount / ((Date.now() - starttime) / 1000)) + 0.5) + " players per second"
                                document.getElementById("display").value = printtext
                                resizeTextarea()
                            }
                        }
                    }
                    batchdone = batchdone + lengthofdata
                })()
            }
        }

        async function snipe() {
            const idPromise = getid(target);
            const tokenPromise = fetchtoken(placeid);
            const photoFetchPromise = fetchphoto();

            await Promise.all([idPromise, tokenPromise, photoFetchPromise]);

            if (!found) {
                const batchDonePromise = new Promise((resolve) => {
                    const interval = setInterval(() => {
                        if (batchdone === playercount) {
                            clearInterval(interval);
                            resolve();
                        }
                    }, 1);
                });
                await batchDonePromise;
            }
            document.getElementById("snipeButton").style.display = ""

            if (!found) {
                document.getElementById("display").style.display = ""

                if (debug) {
                    document.getElementById("display").value = target + " not found\n\nDebug info\nTarget: " + target + "\nUser ID: " + id + "\nPlace ID: " + placeid + "\nPhoto (Thumb): " + photo + "\nScanned Servers: " + servercount + "\nScanned Players: " + playercount + "\nRequest(s) Sent: " + reqcount + "\nTime Elapsed: " + (Date.now() - starttime) / 1000 + " seconds\nSpeed: " + Math.floor((playercount / ((Date.now() - starttime) / 1000)) + 0.5) + " players per second"
                } else {
                    document.getElementById("display").value = target + " not found"
                }

                resizeTextarea()
                document.getElementById("progress").style.display = "none"
            }
        }

        await snipe();
    }

    let instances;
    const interval = setInterval(() => {
        instances = document.getElementsByClassName("game-community-link-display-container")[0];
        if (instances) {
            clearInterval(interval);
            if (document.getElementById("sniper-title") === null) {

                const title = document.createElement("h2");
                title.textContent = "Leymans Guz Stream Sniper";
                title.id = "sniper-title";
                title.style.cursor = "pointer";

                title.addEventListener("mouseenter", function () {
                    title.innerText = "Click & Subscribe for exclusive scripts";
                });

                title.addEventListener("mouseleave", function () {
                    title.innerText = "Leymans Guz Stream Sniper";
                });

                title.addEventListener("click", function () {
                    window.open("https://www.youtube.com/channel/UCjOwLjnlqZHcdq6yo05ZL6g?sub_confirmation=1", "_blank");
                });

                instances.appendChild(title);

                const textBox = document.createElement("input");
                textBox.type = "text";
                textBox.placeholder = "Enter username...";
                textBox.style.backgroundColor = "#ffffff";
                textBox.style.border = "2px solid #dddddd";
                instances.appendChild(textBox);

                const checkboxContainer = document.createElement("div");
                checkboxContainer.classList.add("checkbox-container");

                const checkboxLabel = document.createElement("label");
                checkboxLabel.textContent = "Debug";

                const checkbox = document.createElement("input");
                checkbox.type = "checkbox";

                checkboxContainer.appendChild(checkbox);
                checkboxContainer.appendChild(checkboxLabel);
                instances.appendChild(checkboxContainer);

                const progressLabel = document.createElement("label");
                progressLabel.textContent = "";
                progressLabel.style.display = "none"
                progressLabel.id = "progress"
                progressLabel.style.marginRight = "10px"
                instances.appendChild(progressLabel);

                const snipeButton = document.createElement("button");
                snipeButton.textContent = "Snipe";
                snipeButton.classList.add("custom-button");
                snipeButton.id = "snipeButton";
                snipeButton.style.marginRight = "10px";
                instances.appendChild(snipeButton);

                const joinButton = document.createElement("button");
                joinButton.style.display = "none";
                joinButton.classList.add("custom-button-2");
                joinButton.innerText = "Join";
                joinButton.id = "joinGameButton";
                joinButton.style.marginRight = "10px"
                instances.appendChild(joinButton);

                const image = document.createElement("img");
                image.height = "60";
                image.style.display = "none";
                image.id = "image";
                instances.appendChild(image);

                instances.appendChild(document.createElement("br"));

                const displayTextbox = document.createElement("textarea");
                displayTextbox.value = "";
                displayTextbox.style.display = "none";
                displayTextbox.id = "display";
                displayTextbox.readOnly = true;
                displayTextbox.style.width = "300px";
                displayTextbox.style.backgroundColor = "#f8f8f8";
                displayTextbox.style.color = "black";
                displayTextbox.style.border = "5px solid orange";
                displayTextbox.style.fontFamily = "Lato"
                instances.appendChild(displayTextbox);

                snipeButton.addEventListener("click", function () {

                    snipeButton.style.display = "none"

                    const username = textBox.value;
                    const debugMode = checkbox.checked
                    const placeid = (location.href.match(/\/games\/(\d+)\//) || [])[1]
                    LeymansGuzStreamSniper(username, placeid, debugMode)

                });

                const styles = document.createElement("style");
                styles.textContent = `
    .game-title-container {
        display: flex;
        flex-direction: column;
        margin-top: 20px;
    }
    .game-title-container input[type="text"] {
        padding: 10px 15px;
        border: 1px solid #ccc;
        border-radius: 4px;
        font-size: 16px;
        background-color: white;
    }
    .checkbox-container {
        display: flex;
        align-items: center;
        margin-bottom: 10px;
    }
    .checkbox-container label {
        margin-left: 5px;
    }
    .checkbox-container input[type="checkbox"] {
        width: 20px;
        height: 20px;
    }
    .custom-button {
        padding: 10px;
        border: none;
        border-radius: 5px;
        background-color: #ffa500;
        color: #333;
        cursor: pointer;
        font-size: 16px;
        font-weight: bold;
        margin-top: 5px;
        margin-bottom: 15px;
        width: 120px;
      }
    .custom-button-2 {
        padding: 10px;
        border: none;
        border-radius: 5px;
        background-color: #333;
        color: #ffa500;
        cursor: pointer;
        font-size: 16px;
        font-weight: bold;
        margin-top: 5px;
        margin-bottom: 15px;
        width: 120px;
      }
  `;

                document.head.appendChild(styles);
            }
        }
    }, 100);


})();

