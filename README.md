## MX Player Games Documentation

### MX Player - HTML5 Games Integration

### Integration Steps:

The MXPlayer App opens the game in a web-view. These games are opened by the App in 2 ways:

* Uploading a zip file
* Directly by a url.

In a zip file based approach, the game is downloaded only when the user clicks on a particular game inside the `Games` tab in the app. Also whenever a new version of a game is uploaded, the app re-downloads the game to get the latest build.

A Script has to be included in the game initialisation which basically adds an object `cc` in the window scope. This helps in communication from app to game. Please contact us to get the script.

When the web-view opens a game, it passes an object `gameManager` to the window scope which serves the communication from the game to the app. The object format is:

```
window.gameManager = {
    onGameInit: function() {},
    onGameStart: function() {},
    onGameSettings: function() {},
    onTrack: function() {},
    onError: function() {},
    onCheckRewardedVideoAds: function() {},
    onShowRewardedVideoAds: function() {},
    onGameOver: function() {}
}
```

Let’s go through each of these methods:

### gameManager.onGameInit()

This function returns few common parameters used across all games like:
roomID, userID, gameID, highestScore, gameMode, isFirstOpen

```
try {
    if (typeof gameManager !== 'undefined') {
        const gameConfigString = gameManager.onGameInit()
        const config = JSON.parse(gameConfigString)
        const {userID, gameID, roomID, highestScore, gameMode, isFirstOpen} = config
    }
} catch (e) {
    console.log("Error Parsing Config")
}
```

**Note**: The data received is a stringified JSON

Currently the returned object has the following fields:

```
var config = {
   userID: "1",
   gameID: "2",
   roomID: "3",
   highestScore: 90,
   lastLevel: 0,
   gameMode: "score", // "score" or "win"
   isFirstOpen: true, // true or false
   roomType: "free_room" // "free_room" or "priced_room"
}
```

### gameManager.onGameStart()

This function has to be called by the web-view when the game is ready and set to play, to notify the App that the game is ready.

```
if (typeof gameManager !== 'undefined') {
    try {
        gameManager.onGameStart()
    } catch (e) {
        gameManager.onError(e.stack.toString())
    }
}
```

### gameManager.getGameSettings()

This function is called to get configurable parameters for the game. Many difficulty, powerup, probability etc can be configured to be received through this function. MX Player CMS takes these inputs and passes them onto the game through this function. This way, no code changes are required in the game parameters are to be altered.

```
var defaultConfig = {
    reviveScore: 0,
    reviveEnabled: true,
    reviveLives: 1,
    reviveAdExistsDefault: true,
    autoAd: true,
    noDieScore: 10
}

var loadConfig = function () {
    if (typeof gameManager !== 'undefined') {
        try {
            var gameSettingString = gameManager.getGameSettings()
            var config = JSON.parse(gameSettingString)
            return config
        } catch (e) {
            return defaultConfig
        }
    } else {
        return defaultConfig
    }
}

var config = loadConfig()

module.exports = config
```

**Note**: The data received is a stringified JSON

### gameManager.onTrack()

This function is used when the game wants to track events in the game.

```
if (typeof gameManager !== 'undefined') {
    try {
        var obj = {
            score: score,
            timestamp: timestamp
        }
        var data = JSON.stringify(obj)
        gameManager.onTrack('gamePause', data)
    } catch (e) {
        gameManager.onError(e.stack.toString())
    }
}
```

**Note**: The data passed for tracking has to be a stringified JSON
(Please contact us to get the list of events currently tracked)

### gameManager.onError()

This function is used to send errors that might occur in the game. The webview is automatically closed by the App when this method is called. 

The game developers can contact us to get the error logs.

```
if (typeof gameManager !== 'undefined') {
    try {
        // Game Code
    } catch (e) {
        gameManager.onError(e.stack.toString())
    }
}
```

**Note**: The parameter passed should be a string

### gameManager.onCheckRewardedVideoAds()

This is a method which has to be called by the game whenever it wants to check if an ad can be shown or not. This method checks if an ad can be shown by checking many factors like ad unit, internet connectivity etc.


```
if (typeof gameManager !== 'undefined' && 
    typeof gameManager.onCheckRewardedVideoAds === 'function'
) {
    try {
        gameManager.onCheckRewardedVideoAd('rewardAdsExist')
    } catch (e) {
        gameManager.onError(e.stack.toString())
    }
}
```

The parameter that is passed to this function is a String. 
Since this method is asynchronous, once the App computes if an ad can be shown it emits an event.

```
cc.game.emit('rewardAdsExist')
```
**Note**: This is not to be implemented in game code

**Note**: The same event name what was passed to `onCheckRewardedAds` method

The game should be listening to this event by the following syntax

```
cc.game.on('rewardAdsExist', this.onRewardedAdsCheck)
```

The 2nd parameter provided in this function should be a function and will be called when the App emits the event.

Inside the method the implementation would look something like this:

```
onRewardedAdsCheck: function (result) {
    if (result.statyus === 0) {
        // Ad can be shown
    } else {
        // Ad cannot be shown
    }
}
```

**Tips**: Since gameManager.onCheckRewardedAds cannot be tested locally, for testing purposes, from your web console type:

```
cc.game.emit('rewardAdsExist', {status: 0}) // To test flow: ad exists

cc.game.emit('rewardAdsExist', {status: 1}) // To test flow: ad does not exists
```

This will call the game method (onRewardedAdsCheck in the above example). 

### gameManager.onShowRewardedVideoAds()

This method starts the video ad on top of the game. The game should pause at this time and all sound effects should be stopped for the ad to play.


```
if (typeof gameManager !== 'undefined' && 
    typeof gameManager.onShowRewardedVideoAds === 'function'
) {
    try {
        gameManager.onShowRewardedVideoAds('rewardLife', null)
    } catch (e) {
        gameManager.onError(e.stack.toString())
    }
}
```

The first parameter that is passed to this function is a String. The second parameter with value `null` is mandatory.
Since this method is asynchronous, once the ad is played or the ad is quitted (by clicking on close icon on the ad screen), the App emits an event to the game.

```
cc.game.emit('onAdPlayed')
```

**Note**: This is not to be implemented in game code

**Note**: The same event name what was passed to `onShowRewardedVideoAds` method

The game should be listening to this event by the following syntax

```
cc.game.on('onAdPlayed', this.adPlayed)
```

The 2nd parameter provided in this function should be a function and will be called when the App emits the event.

Inside the method the implementation would look something like this:

```
adPlayed: function (result) {
    if (result.statyus === 0) {
        // Ad can be shown
    } else {
        // Ad cannot be shown
    }
}
```

**Tips**: Since gameManager.onShowRewardedVideoAds cannot be tested locally, for testing purposes, from your web console type:

```
cc.game.emit('onAdPlayed', {status: 0}) // To test flow: ad exists

cc.game.emit('onAdPlayed', {status: 1}) // To test flow: ad does not exists
```

This will call the game method (onAdPlayed in the above example).


### gameManager.onGameOver()

This function is used when the game is over and needs to be closed. The game needs to send parameters like score, highScore etc. The parameters score and highScore need to be integers.

```
if (typeof gameManager !== 'undefined') {
    var obj = {
        gameID: String(cc.sys.localStorage.getItem('JuegoJumpgameId')),
        roomID: String(cc.sys.localStorage.getItem('JuegoJumproomId')),
        userID: String(cc.sys.localStorage.getItem('JuegoJumpgameId')),
        score: this.score,
        highScore: highScore,
        info: encryption.getInfo(this.totalScore, this.gameplayTimeInSecond, reviveCount)
    }
    try {
        var score = JSON.stringify(obj)
        gameManager.onGameOver(score)
    } catch (e) {
        gameManager.onError(e.stack.toString())
    }
}
```

### Base Analytics Events

| Event Name                      	| Parameters   	| Possible Values             	| Description                                                                                       	|
|---------------------------------	|--------------	|-----------------------------	|---------------------------------------------------------------------------------------------------	|
| gameStart                       	| userID       	| N/A                         	| Received from onGameInit()                                                                        	|
| On Game Start                   	| gameID       	| N/A                         	| Received from onGameInit()                                                                        	|
|                                 	| roomID       	| N/A                         	| Received from onGameInit()                                                                        	|
|                                 	| startType    	| “first” / “new” / “restart” 	| first - First Session -> onGameInit().isFirstOpen / new - Returning user / restart - Restart Game 	|
|                                 	|              	|                             	|                                                                                                   	|
| gamePause                       	| userID       	| N/A                         	| Received from onGameInit()                                                                        	|
| On play after paused            	| gameID       	| N/A                         	| Received from onGameInit()                                                                        	|
|                                 	| roomID       	| N/A                         	| Received from onGameInit()                                                                        	|
|                                 	| currentTime  	| Type - Integer              	| No: of seconds paused                                                                             	|
|                                 	|              	|                             	|                                                                                                   	|
| gameExit                        	| userID       	| N/A                         	| Received from onGameInit()                                                                        	|
| On Game Over                    	| gameID       	| N/A                         	| Received from onGameInit()                                                                        	|
|                                 	| roomID       	| N/A                         	| Received from onGameInit()                                                                        	|
|                                 	| currentScore 	| Type - Integer              	| Game Score                                                                                        	|
|                                 	| highScore    	| Type - Integer              	| Received from onGameInit()                                                                        	|
|                                 	| playTime     	| Type - Integer              	| No: of seconds played                                                                             	|
|                                 	| adClaimed    	|  0/1                        	| 0 - Ad not claimed 1 - Ad claimed                                                                 	|
|                                 	|              	|                             	|                                                                                                   	|
| gameAdShown                     	| userID       	| N/A                         	| Received from onGameInit()                                                                        	|
| On Watch Ad screen shown        	| gameID       	| N/A                         	| Received from onGameInit()                                                                        	|
|                                 	| roomID       	| N/A                         	| Received from onGameInit()                                                                        	|
|                                 	|              	|                             	|                                                                                                   	|
| gameAdClicked                   	| userID       	| N/A                         	| Received from onGameInit()                                                                        	|
|                                 	| gameID       	| N/A                         	| Received from onGameInit()                                                                        	|
|                                 	| roomID       	| N/A                         	| Received from onGameInit()                                                                        	|
|                                 	| autoPlayed   	| 0/1                         	| If autoplay of Ad is enabled. 0 - Not Autoplayed  1 - Autoplayed                                  	|
|                                 	|              	|                             	|                                                                                                   	|
| gameAdClaimed                   	| userID       	| N/A                         	| Received from onGameInit()                                                                        	|
| On completion or quitting of Ad 	| gameId       	| N/A                         	| Received from onGameInit()                                                                        	|
|                                 	| roomId       	| N/A                         	| Received from onGameInit()                                                                        	|
|                                 	| autoPlayed   	| 0/1                         	| If autoplay of Ad is enabled. 0 - Not Autoplayed  1 - Autoplayed                                  	|


### Game Manager Library SDK

This library would actually facilitate all the common functionalities required across all games. For start, the current version would contain the following functionalities:

* Add a polyfill for `cc` namespace required for games made on other frameworks like PixieJS and PhaserJS.
* iOS App cannot expose the `gameManager` object for the games. So instead this library would include functionalities which would enable iOS communication with iOS App without any change in game code.
* All games require the sticky banners to be shown. This library wraps all the adRelated logic on one place and all the game has to do is define the position where the sticky ad has to be shown
