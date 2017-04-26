---
layout: post
title: How To Load/Save Game State in XNA
date: '2010-11-25 12:10:05'
---

There seems to be a lack of articles about how to load and save Game state with XNA.
After an hour or so researching and trying different things I managed to get a really
basic Save/Load working in my game by doing the following.

Please note so far this has only been tested on Windows with XNA 4.0 it may or may
not work for Xbox/Phone/Zune, I believe the StorageDevice/Container objects I used
have been changed recently so I doubt it will work with older versions of XNA.

In your games main constructor add the following line

this.Components.Add(new GamerServicesComponent(this));

This gives us the ability to check if the guide interface is showing on the Xbox360
as if it is we need to cancel saving as the user will be unable to pick a storage
container. This is not needed for saving on PC but I put it in to try to make life
easier when I get round to porting to Xbox.

In the class you want to do your Saving and Loading create the following members
<pre class="brush: csharp; toolbar: false;">      StorageDevice device;
      string containerName="MyGamesStorage";
      string filename = "mysave.sav";
      [Serializable]
      public struct SaveGame
      {
         public Vector2 PlayerPosition;
      }</pre>
The SaveGame struct contains the data we are going to save, in this case just a Vector2
but you can add as much to this as you need.

Add the following methods
<pre class="brush: csharp; toolbar: false;">        private void InitiateSave()
        {
            if (!Guide.IsVisible)
            {
                device = null;
                StorageDevice.BeginShowSelector(PlayerIndex.One, this.SaveToDevice, null);
            }
        }

        void SaveToDevice(IAsyncResult result)
        {
            device = StorageDevice.EndShowSelector(result);
            if (device != null &amp;&amp; device.IsConnected)
            {
                SaveGame SaveData = new SaveGame()
                {
                    PlayerPosition = gamePlayer.Position,
                };
                IAsyncResult r = device.BeginOpenContainer(containerName, null, null);
                result.AsyncWaitHandle.WaitOne();
                StorageContainer container = device.EndOpenContainer(r);
                if (container.FileExists(filename))
                    container.DeleteFile(filename);
                Stream stream = container.CreateFile(filename);
                XmlSerializer serializer = new XmlSerializer(typeof(SaveGame));
                serializer.Serialize(stream, SaveData);
                stream.Close();
                container.Dispose();
                result.AsyncWaitHandle.Close();
            }
        }</pre>
Then just make a call to InitiateSave when ever you want to save your game.

The load code is very similar...
<pre class="brush: csharp; toolbar: false;">        private void InitiateLoad()
        {
            if (!Guide.IsVisible)
            {
                device = null;
                StorageDevice.BeginShowSelector(PlayerIndex.One, this.LoadFromDevice, null);
            }
        }

        void LoadFromDevice(IAsyncResult result)
        {
            device = StorageDevice.EndShowSelector(result);
            IAsyncResult r = device.BeginOpenContainer(containerName, null, null);
            result.AsyncWaitHandle.WaitOne();
            StorageContainer container = device.EndOpenContainer(r);
            result.AsyncWaitHandle.Close();
            if (container.FileExists(filename))
            {
                Stream stream = container.OpenFile(filename, FileMode.Open);
                XmlSerializer serializer = new XmlSerializer(typeof(SaveGame));
                SaveGame SaveData = (SaveGame)serializer.Deserialize(stream);
                stream.Close();
                container.Dispose();
                //Update the game based on the save game file
                gamePlayer.Position = SaveData.PlayerPosition;
            }
        }</pre>
Looking back at that, its probably worth refactoring the InitiateLoad/InitiateSave
methods to be a single InitiateDevice method passing in a delegate for the Load/Save
method you want to call once the device is initialized.