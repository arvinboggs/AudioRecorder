# Blazor Audio Recorder.

Add audio recording features to your Blazor Webassembly app by harnessing JavaScript interop.

> Please view the live demo at [https://arvinboggs.github.io/AudioRecorderLive](https://arvinboggs.github.io/AudioRecorderLive)

> Project page: [https://github.com/arvinboggs/AudioRecorder](https://github.com/arvinboggs/AudioRecorder)

1. Open (or create) a Blazor WebAssembly project.

2. Create a JavaScript file, named `AudioRecorder.js`, inside the `wwwroot/scripts` folder. Create the `scripts` folder if it does not exist yet.
    > At the time of writing this article, Blazor has several things that it cannot do out of the box. One of these things is recording audio. Therefore, we will use JavaScript interop to achieve our goal.

3. Put the initial content of `AudioRecorder.js`:

    ``` javascript
    // AudioRecorder.js
    var BlazorAudioRecorder = {};
    (function () {

    })();
    ```
    We will be needing Blazor to call our JavaScript functions later and the only JavaScript objects that are visible to Blazor are the global-level ones. So we declared `BlazorAudioRecorder` as a global variable. This variable will be the parent object for the functions that we will be creating later.

4. Add code to start the recording.
    ``` javascript
    var mStream;
    var mMediaRecorder;

    BlazorAudioRecorder.StartRecord = async function () {
        mStream = await navigator.mediaDevices.getUserMedia({ audio: true });
        mMediaRecorder = new MediaRecorder(mStream);
        mMediaRecorder.start();
    };
    ```
    The preceding code will start the recording. But, by itself, will not be able to output anything useful.

5. Add the function to capture data while recording. Insert this code before the `mMediaRecorder.start()` line.
    ``` javascript
    mMediaRecorder.addEventListener('dataavailable', vEvent => {
    mAudioChunks.push(vEvent.data);
    });
    ```

6. Before stopping recording, we should be able to capture the final bits of data and convert the entire recording into a usable audio file.
    ``` javascript
    mMediaRecorder.addEventListener('stop', () => {
        var pAudioBlob = new Blob(mAudioChunks, { type: "audio/mp3;" });
        var pAudioUrl = URL.createObjectURL(pAudioBlob);
    });
    ```

7. Add the function to stop the recording.
    ``` javascript
    BlazorAudioRecorder.StopRecord = function () {
        mMediaRecorder.stop();
        mStream.getTracks().forEach(pTrack => pTrack.stop());
    };
    ```

8. It's time to edit our `Index.razor` file. Add the buttons.
    ``` html
    <!-- Index.razor -->
    <button>Start Record</button>
    <button>Pause</button>
    <button>Resume</button>
    <button>Stop</button>
    <button>Download Audio</button>
    ```

9. Then inject `IJSRuntime` at the top of the page.
    ``` razor
    @inject IJSRuntime mJS
    ```

10. Add a `Click` event handler to the **Start Record** button. The `BlazorAudioRecorder.StartRecord` is from the JavaScript file that we created earlier.

    ``` html
    <button @onclick="butRecordAudioStart_Click">Start Record</button>
    ```

    ``` c#
    @code {
    void butRecordAudioStart_Click()
        {
            mJS.InvokeVoidAsync("BlazorAudioRecorder.StartRecord");
        }
    }
    ```

11. Likewise, add a `Click` event handler to each of the remaining buttons. Each event handler is just a line of code that invokes its respective JavaScript function.

12. In our JavaScript file, inside the `MediaRecorder` `stop` event, we need to find a way for the JavaScript to pass the audio URL back to our Blazor code. First, we need to pass an instance of our Razor class to JavaScript. The following code will call the `Initialize` (which we will code later) function in our JavaScript file passing the current instance of our Razor class as the parameter.
    ``` c#
    protected override async Task OnInitializedAsync()
    {
        await base.OnInitializedAsync();
        await mJS.InvokeVoidAsync("BlazorAudioRecorder.Initialize", DotNetObjectReference.Create(this));
    }
    ```

13. Implement `BlazorAudioRecorder.Initialize` in our JavaScript file.
    ``` javascript
    // AudioRecorder.js
    var mCaller;

    BlazorAudioRecorder.Initialize = function (vCaller) {
        mCaller = vCaller;
    };
    ```

14. Now that we have an instance of our Razor class (in a form of `DotNetObjectReference`), we can now pass the audio URL to our Blazor code.
    ``` javascript
    mMediaRecorder.addEventListener('stop', () => {
        var pAudioBlob = new Blob(mAudioChunks, { type: "audio/mp3;" });
        var pAudioUrl = URL.createObjectURL(pAudioBlob);
        mCaller.invokeMethodAsync('OnAudioUrl', pAudioUrl);
    });
    ```
    In the preceding code, a call was made to `OnAudioUrl` (which will code in the next step) passing the audio URL.

15. Implement the `OnAudioUrl` function in our Razor file. The function must be `public` and must be decorated with `JSInvokable` attribute so that it can be called from JavaScript.

    ``` c#
    // Index.razor
    string mUrl;

    [JSInvokable]
    public async Task OnAudioUrl(string vUrl)
    {
        mUrl = vUrl;
        await InvokeAsync(() => StateHasChanged());
    }
    ```

16. Pass the audio URL to the `audio` HTML element. By doing this, the user now playback the recording.
    ``` html
    <audio controls autoplay src=@mUrl></audio>
    ```

17. For the **Download Audio** button's `click` event, make a call to JavaScript.
    ``` c#
    void butDownloadBlob_Click()
    {
        mJS.InvokeVoidAsync("BlazorAudioRecorder.DownloadBlob", mUrl, "MyRecording.mp3");
    }
    ```

18. Implement the `DownloadBlob` function in our JavaScript file. Note that in this "download", everything happens in the browser and no server is involved.
    ``` javascript
    BlazorAudioRecorder.DownloadBlob = function (vUrl, vName) {
        // Create a link element
        const link = document.createElement("a");

        // Set the link's href to point to the Blob URL
        link.href = vUrl;
        link.download = vName;

        // Append link to the body
        document.body.appendChild(link);

        // Dispatch click event on the link
        // This is necessary as link.click() does not work on the latest firefox
        link.dispatchEvent(
            new MouseEvent('click', {
                bubbles: true,
                cancelable: true,
                view: window
            })
        );

        // Remove the link from the body
        document.body.removeChild(link);
    };
    ```