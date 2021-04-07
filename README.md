## MX Player - HTML5 Games Integration

### Integration Steps

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
    getGameSettings: function() {},
    onTrack: function() {},
    onError: function() {},
    onCheckRewardedVideoAds: function() {},
    onShowRewardedVideoAds: function() {},
    onGameOver: function() {}
}
```

#### gameManager.onGameInit()

This function returns few common parameters used across all games like:
roomId, userId, gameId, highestScore, gameMode, isFirstOpen

```
try {
    if (typeof gameManager !== 'undefined') {
        const gameConfigString = gameManager.onGameInit()
        const config = JSON.parse(gameConfigString)
        const {userId, gameId, roomId, highestScore, gameMode, isFirstOpen} = config
    }
} catch (e) {
    console.log("Error Parsing Config")
}
```

**Note**: The data received is a stringified JSON

Currently the returned object has the following fields:

```
var config = {
   userId: "1",
   gameId: "2",
   roomId: "3",
   highestScore: 90,
   lastLevel: 0, // user's current level
   gameMode: "score", // "score" or "win"
   isFirstOpen: true, // true or false
   roomType: "free_room" // "free_room" or "priced_room"
}
```

#### gameManager.onGameStart()

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

#### gameManager.getGameSettings()

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

#### gameManager.onTrack()

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

#### gameManager.onError()

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

#### gameManager.onCheckRewardedVideoAds()

This is a method which has to be called by the game whenever it wants to check if an ad can be shown or not. This method checks if an ad can be shown by checking many factors like ad unit, internet connectivity etc.


```
if (typeof gameManager !== 'undefined' && 
    typeof gameManager.onCheckRewardedVideoAds === 'function'
) {
    try {
        gameManager.onCheckRewardedVideoAds('rewardAdsExist')
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
    if (result.status === 0) {
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

#### gameManager.onShowRewardedVideoAds()

This method starts the video ad on top of the game. The game should pause at this time and all sound effects should be stopped for the ad to play.


```
if (typeof gameManager !== 'undefined' && 
    typeof gameManager.onShowRewardedVideoAds === 'function'
) {
    try {
        gameManager.onShowRewardedVideoAds('onAdPlayed', null)
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
    if (result.status === 0) {
        // completely watched the ad. can give reward
    } else {
        // did not watch the ad completely. no reward
    }
}
```

**Tips**: Since gameManager.onShowRewardedVideoAds cannot be tested locally, for testing purposes, from your web console type:

```
cc.game.emit('onAdPlayed', {status: 0}) // To test flow: ad exists

cc.game.emit('onAdPlayed', {status: 1}) // To test flow: ad does not exists
```

This will call the game method (onAdPlayed in the above example).


#### gameManager.onGameOver()

This function is used when the game is over and needs to be closed. The game needs to send parameters like score, highScore etc. The parameters score and highScore need to be integers.

```
if (typeof gameManager !== 'undefined') {
    var obj = {
        gameID: String(cc.sys.localStorage.getItem('gameID')),
        roomID: String(cc.sys.localStorage.getItem('roomID')),
        userID: String(cc.sys.localStorage.getItem('userID')),
        score: this.score,
        highScore: highScore,
        lastLevel: lastLevel,
        info: encryption.getInfo(lastLevel, this.gameplayTimeInSecond, reviveCount), // only for level based games
        info: encryption.getInfo(this.score, this.gameplayTimeInSecond, reviveCount) // for other games
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

| Event Name                      	| Parameters   	| Possible Values             	| Description                                                                                                        	|
|---------------------------------	|--------------	|-----------------------------	|--------------------------------------------------------------------------------------------------------------------	|
| gameStart                       	| userID       	| N/A                         	| Received from onGameInit()                                                                                         	|
| On Game Start                   	| gameID       	| N/A                         	| Received from onGameInit()                                                                                         	|
|                                 	| roomID       	| N/A                         	| Received from onGameInit()                                                                                         	|
|                                 	| startType    	| “first” / “new” / “restart” 	| first - First Session -> onGameInit().isFirstOpen / new - Returning user / restart - Restart Game                  	|
|                                 	|              	|                             	|                                                                                                                    	|
| gamePause                       	| userID       	| N/A                         	| Received from onGameInit()                                                                                         	|
| On play after paused            	| gameID       	| N/A                         	| Received from onGameInit()                                                                                         	|
|                                 	| roomID       	| N/A                         	| Received from onGameInit()                                                                                         	|
|                                 	| currentTime  	| Type - Integer              	| No: of seconds paused                                                                                              	|
|                                 	|              	|                             	|                                                                                                                    	|
| gameExit                        	| userID       	| N/A                         	| Received from onGameInit()                                                                                         	|
| On Game Over                    	| gameID       	| N/A                         	| Received from onGameInit()                                                                                         	|
|                                 	| roomID       	| N/A                         	| Received from onGameInit()                                                                                         	|
|                                 	| currentScore 	| Type - Integer              	| Game Score                                                                                                         	|
|                                 	| highScore    	| Type - Integer              	| Received from onGameInit()                                                                                         	|
|                                 	| playTime     	| Type - Integer              	| No: of seconds played                                                                                              	 |
|                                 	| lastLevel     | Type - Integer              	| For level based games. Indicates levels that the user has completed.                                                  |
|                                 	| adGameStartOpportunity |  Type - Integer      | No. of times option to watch ad is shown at the start of the game                                                                  |
|                                 	| adGameStartShown    	 |  Type - Integer      | No. of times ad is shown at the start of the game                                                            |
|                                 	| adGameStartClaimed     |  Type - Integer      | No. of times ad is successfully claimed at start of the game                                                                        |
|                                 	| adGameEndOpportunity   |  Type - Integer      | No. of times option to watch ad is shown at the end of the game                                                                |
|                                 	| adGameEndShown    	 |  Type - Integer      | No. of times ad is shown at end of the game                                                              |
|                                 	| adGameEndClaimed    	 |  Type - Integer      | No. of times ad is successfully claimed at the end of the game                                                                      |
|                                 	| adGamePowerupClaimed   |  Type - Integer      | No. of times ad is claimed to earn a powerup in mid of the game                                                        |
|                                 	|              	|                             	|                                                                                                                    	|
| gameAdShown                     	| userID       	| N/A                         	| Received from onGameInit()                                                                                         	|
| On Watch Ad screen shown        	| gameID       	| N/A                         	| Received from onGameInit()                                                                                         	|
|                                 	| roomID       	| N/A                         	| Received from onGameInit()                                                                                         	|
|                                 	| position     	| "start"/"mid"/"end"         	| start for ads shown at the beginning of the game, mid for ad shown during the game and end for ad shown at the end 	|
|                                 	|              	|                             	|                                                                                                                    	|
| gameAdClicked                   	| userID       	| N/A                         	| Received from onGameInit()                                                                                         	|
|                                 	| gameID       	| N/A                         	| Received from onGameInit()                                                                                         	|
|                                 	| roomID       	| N/A                         	| Received from onGameInit()                                                                                         	|
|                                 	| autoPlayed   	| 0/1                         	| If autoplay of Ad is enabled. 0 - Not Autoplayed  1 - Autoplayed                                                   	|
|                                 	| position     	| "start"/"mid"/"end"         	| start for ads shown at the beginning of the game, mid for ad shown during the game and end for ad shown at the end 	|
|                                 	|              	|                             	|                                                                                                                    	|
| gameAdClaimed                   	| userID       	| N/A                         	| Received from onGameInit()                                                                                         	|
| On completion or quitting of Ad 	| gameID       	| N/A                         	| Received from onGameInit()                                                                                         	|
|                                 	| roomID       	| N/A                         	| Received from onGameInit()                                                                                         	|
|                                 	| autoPlayed   	| 0/1                         	| If autoplay of Ad is enabled. 0 - Not Autoplayed  1 - Autoplayed                                                   	|
|                                 	| position     	| "start"/"mid"/"end"         	| start for ads shown at the beginning of the game, mid for ad shown during the game and end for ad shown at the end 	|

### Sticky Ads

For integration of Sticky Ads the following steps has to be done:

* Add the following parameter in the CMS config parameters
    * `stickyBannersEnabled`- a boolean which indicates whether to enable sticky ads or not.
       
    ```
    "stickyBannersEnabled": true
    ```

* After the game is loaded, use the following snippet of code to show the sticky ads:
    
    ```
    if (config.stickyBannersEnabled === true && typeof gameManager !== 'undefined' && typeof gameManager.showStickyAds === 'function') {
            cc.game.on('adShown', () => {
                /* do something when ad is shown after showStickyAds call */
            })
            cc.game.on('adNotShown', () => {
                /* do something when ad is not shown after showStickyAds call */
            })
            gameManager.showStickyAds('bottom')
    }
    ```
    When `showStickyAds` function is called with a position parameter, it shows an ad at that position which can either be `top` or `bottom`.

    Note: App sends the event `adShown` or `adNotShown` when `showStickyAds` function is called. You can make minor position adjustments in the Game when these events are received. Ad size mostly is 320 X 50.

    To hide sticky ads at anytime use this function:

    ```
    gameManager.hideStickyAds()
    ```

### Notify Game Events
`Only for Android` The host app will call the following js function(s) when the corresponding event is received.

Events:
    
   * `backPressed`- Device back button was pressed
    
   ```
   cc.game.on('backPressed', function(result) {});
   ```
   
   * `screenOn`- Device screen turned on
    
   ```
   cc.game.on('screenOn', function(result) {});
   ```
   
   * `screenOff`- Device screen turned off
    
   ```
   cc.game.on('screenOff', function(result) {});
   ```
   
   * `userPresent`- User unlocks device
    
   ```
   cc.game.on('userPresent', function(result) {});
   ```
   
   * `pagePause`- Activity onPause
    
   ```
   cc.game.on('pagePause', function(result) {});
   ```
   
   * `pageResume`- Activity onResume 
    
   ```
   cc.game.on('pageResume', function(result) {});
   ```
   
   * `homePressed`- Device home button was pressed
    
   ```
   cc.game.on('homePressed', function(result) {});
   ```
   
   * `recentPressed`- Device recentApps button was pressed
    
   ```
   cc.game.on('recentPressed', function(result) {});
   ```
   
**Note**: `result` is in json format. In the above events, there is no need to check the `result`.
