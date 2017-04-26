---
layout: post
title: XNA Animating A Sprite between 2 Vector2 Structures
date: '2011-01-09 08:09:02'
---

<p>
If you have a sprite on the screen that you want to smoothly animate from one vector2
to another then Vector2.Lerp is a nice simple way to achieve this. 
</p>
<p>
Start off by adding the following variables to your game class 
</p>
<pre class="brush: csharp; toolbar: false;">
        Texture2D BallTexture;
        Vector2 CurrentPosition = new Vector2(100, 100);
        Vector2 StartPosition = new Vector2(100, 100);
        Vector2 EndPosition = new Vector2(500, 120);
        TimeSpan AnimationLengthSeconds = TimeSpan.FromSeconds(4);
</pre>
<p>
In the LoadContent method load your Texture2D in to your Texture2D object in this
case BallTexture 
</p>
<pre class="brush: csharp; toolbar: false;">
        protected override void LoadContent()
        {
            spriteBatch = new SpriteBatch(GraphicsDevice);
            BallTexture = Content.Load&lt;Texture2D&gt;("ball");
        }
</pre>
<p>
In your Draw method draw the Texture2D to the screen using CurrentPosition for the
Vector2 position. 
</p>
<pre class="brush: csharp; toolbar: false;">
	protected override void Draw(GameTime gameTime)
        {
            GraphicsDevice.Clear(Color.White);
            spriteBatch.Begin();
            spriteBatch.Draw(BallTexture, CurrentPosition, Color.White);
            spriteBatch.End();
            base.Draw(gameTime);
        }
</pre>
<p>
If you run the project now you will see the Texture drawn to the screen at 100,100.
Now to animate it we need to add the following code to the Update method. 
</p>
<pre class="brush: csharp; toolbar: false;">
        protected override void Update(GameTime gameTime)
        {
            if (GamePad.GetState(PlayerIndex.One).Buttons.Back == ButtonState.Pressed)
                this.Exit();
            if (AnimationPosition &lt; AnimationLengthSeconds)
            {
                AnimationPosition += gameTime.ElapsedGameTime;
                if (AnimationPosition >= AnimationLengthSeconds)
                    AnimationPosition = AnimationLengthSeconds;
                var LerpAmount = (float)AnimationPosition.Ticks / AnimationLengthSeconds.Ticks;
                CurrentPosition = Vector2.Lerp(StartPosition, EndPosition, LerpAmount);
            }  
            base.Update(gameTime);
        }
</pre>
<p>
This will change the CurrentPosition on every update to be closer to the EndPosition,
the amount it moves the CurrentPostion by depends on the duration you set for the
animation in AnimationLengthSeconds and the elapsed time of the animation stored in
AnimationPosition. 
</p>