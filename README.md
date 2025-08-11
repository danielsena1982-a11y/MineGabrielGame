
// Simple Phaser 3 game: MineGabriel Game
const config = {
  type: Phaser.AUTO,
  parent: 'game-container',
  width: window.innerWidth,
  height: window.innerHeight,
  scale: {
    mode: Phaser.Scale.FIT,
    autoCenter: Phaser.Scale.CENTER_BOTH
  },
  physics: {
    default: 'arcade',
    arcade: { gravity: { y: 900 }, debug: false }
  },
  scene: [BootScene, MenuScene, PlayScene, UIScene]
};

let game = new Phaser.Game(config);

// BootScene loads assets
function BootScene() {
  Phaser.Scene.call(this, { key: 'BootScene' });
}
BootScene.prototype = Object.create(Phaser.Scene.prototype);
BootScene.prototype.constructor = BootScene;
BootScene.prototype.preload = function() {
  this.load.image('char_a', 'assets/char_a.png');
  this.load.image('char_b', 'assets/char_b.png');
  this.load.image('block_g', 'assets/block_grass.png');
  this.load.image('block_s', 'assets/block_stone.png');
  this.load.image('block_color', 'assets/block_color.png');
  this.load.image('btn', 'assets/btn.png');
  this.load.audio('jump','assets/jump.wav');
  this.load.audio('hit','assets/hit.wav');
  this.load.audio('win','assets/win.wav');
  this.load.audio('music','assets/music.wav');
};
BootScene.prototype.create = function() {
  this.scene.start('MenuScene');
};

// MenuScene: select character
function MenuScene() {
  Phaser.Scene.call(this, { key: 'MenuScene' });
}
MenuScene.prototype = Object.create(Phaser.Scene.prototype);
MenuScene.prototype.constructor = MenuScene;
MenuScene.prototype.create = function() {
  const w = this.scale.width, h = this.scale.height;
  this.add.text(w/2, h*0.18, 'MineGabriel Game', { fontSize: '32px', fill:'#fff' }).setOrigin(0.5);
  this.add.text(w/2, h*0.28, 'Escolha seu personagem', { fontSize: '20px', fill:'#fff' }).setOrigin(0.5);

  const a = this.add.image(w*0.35, h*0.55, 'char_a').setScale(3);
  const b = this.add.image(w*0.65, h*0.55, 'char_b').setScale(3);
  a.setInteractive(); b.setInteractive();

  a.on('pointerup', ()=> { this.registry.set('player','a'); this.scene.start('PlayScene'); });
  b.on('pointerup', ()=> { this.registry.set('player','b'); this.scene.start('PlayScene'); });

  // auto full screen on user gesture (menu click)
  this.input.once('pointerdown', () => {
    if (this.scale.isFullscreen) return;
    this.scale.startFullscreen();
  });
};

// PlayScene: main game
function PlayScene() {
  Phaser.Scene.call(this, { key: 'PlayScene' });
  this.level = 1;
  this.score = 0;
  this.best = 0;
}
PlayScene.prototype = Object.create(Phaser.Scene.prototype);
PlayScene.prototype.constructor = PlayScene;
PlayScene.prototype.create = function() {
  const w = this.scale.width, h = this.scale.height;
  this.best = parseInt(localStorage.getItem('mg_best')||'0');
  this.registry.set('best', this.best);
  this.registry.set('score', 0);

  // music
  this.music = this.sound.add('music', { loop:true, volume:0.3 });
  this.music.play();

  // platforms group (using blocks)
  this.platforms = this.physics.add.staticGroup();
  // ground
  for (let i=0;i<Math.ceil(w/32)+4;i++){
    let x = i*32;
    this.platforms.create(x, h-32, 'block_g').setScale(1).refreshBody();
  }

  // player
  const playerKey = (this.registry.get('player')==='b')?'char_b':'char_a';
  this.player = this.physics.add.sprite(100, h-120, playerKey).setScale(2);
  this.player.setCollideWorldBounds(true);
  this.player.setSize(16,28).setOffset(8,4);
  this.physics.add.collider(this.player, this.platforms);

  // obstacles group
  this.obstacles = this.physics.add.group();
  this.physics.add.collider(this.obstacles, this.platforms);
  this.physics.add.overlap(this.player, this.obstacles, this.hitObstacle, null, this);

  // goal
  this.goal = this.physics.add.staticSprite(w-80, h-120, 'block_color').setScale(1.2);
  this.physics.add.overlap(this.player, this.goal, ()=>{ this.winLevel(); }, null, this);

  // input
  this.cursors = this.input.keyboard.createCursorKeys();
  this.keyA = this.input.keyboard.addKey(Phaser.Input.Keyboard.KeyCodes.A);
  this.keyD = this.input.keyboard.addKey(Phaser.Input.Keyboard.KeyCodes.D);
  this.keySpace = this.input.keyboard.addKey(Phaser.Input.Keyboard.KeyCodes.SPACE);

  // touch buttons (link with DOM buttons)
  const leftBtn = document.getElementById('btn-left');
  const rightBtn = document.getElementById('btn-right');
  const jumpBtn = document.getElementById('btn-jump');
  const runBtn = document.getElementById('btn-run');

  let leftDown=false, rightDown=false, runDown=false, jumpPressed=false;
  leftBtn.addEventListener('pointerdown', ()=> leftDown=true);
  leftBtn.addEventListener('pointerup', ()=> leftDown=false);
  leftBtn.addEventListener('touchstart', ()=> leftDown=true);
  leftBtn.addEventListener('touchend', ()=> leftDown=false);

  rightBtn.addEventListener('pointerdown', ()=> rightDown=true);
  rightBtn.addEventListener('pointerup', ()=> rightDown=false);
  rightBtn.addEventListener('touchstart', ()=> rightDown=true);
  rightBtn.addEventListener('touchend', ()=> rightDown=false);

  runBtn.addEventListener('pointerdown', ()=> runDown=true);
  runBtn.addEventListener('pointerup', ()=> runDown=false);
  runBtn.addEventListener('touchstart', ()=> runDown=true);
  runBtn.addEventListener('touchend', ()=> runDown=false);

  jumpBtn.addEventListener('pointerdown', ()=> { if (this.player.body.touching.down) { this.player.setVelocityY(-420); this.sound.play('jump'); }});
  jumpBtn.addEventListener('touchstart', ()=> { if (this.player.body.touching.down) { this.player.setVelocityY(-420); this.sound.play('jump'); }});

  this.leftDown = ()=> leftDown;
  this.rightDown = ()=> rightDown;
  this.runDown = ()=> runDown;

  // HUD
  this.scoreText = this.add.text(16, 60, 'Score: 0', { fontSize:'18px', fill:'#fff' }).setScrollFactor(0);
  this.bestText = this.add.text(this.scale.width-10, 60, 'Best: '+this.best, { fontSize:'18px', fill:'#fff' }).setOrigin(1,0).setScrollFactor(0);

  // spawn obstacles timer (depends on level)
  this.obstacleTimer = this.time.addEvent({ delay: 1500, callback: this.spawnObstacle, callbackScope: this, loop:true });

  // speed factor per level
  this.setLevel(this.level);
};

PlayScene.prototype.setLevel = function(l) {
  this.level = l;
  if (l===1) { this.gameSpeed = 120; this.obstacleTimer.delay = 1500; }
  if (l===2) { this.gameSpeed = 200; this.obstacleTimer.delay = 1000; }
  if (l===3) { this.gameSpeed = 320; this.obstacleTimer.delay = 650; }
  this.add.text(10, this.scale.height-60, 'Fase '+l, { fontSize:'16px', fill:'#fff' }).setScrollFactor(0);
};

PlayScene.prototype.spawnObstacle = function() {
  let h = this.scale.height;
  let x = this.scale.width + 40;
  let y = h-64 - (Math.random()>0.6?32:0);
  let sprite = Math.random()>0.5 ? 'block_s' : 'block_g';
  let ob = this.obstacles.create(x, y, sprite);
  ob.setVelocityX(-this.gameSpeed);
  ob.setImmovable(true);
  ob.setCollideWorldBounds(false);
  ob.setSize(32,32);
  ob.setScale(1);
  ob.checkWorldBounds = true;
  ob.outOfBoundsKill = true;
};

PlayScene.prototype.hitObstacle = function(player, ob) {
  // lose and restart level
  this.sound.play('hit');
  this.scene.pause();
  this.music.stop();
  this.scene.launch('UIScene', { win: false, score: this.registry.get('score'), restartTo: 1 });
};

PlayScene.prototype.winLevel = function() {
  this.sound.play('win');
  this.music.stop();
  let sc = this.registry.get('score');
  if (this.level < 3) {
    this.level++;
    // restart same scene with next level
    this.scene.restart({ nextLevel: this.level });
  } else {
    // final win
    // update best
    let best = parseInt(localStorage.getItem('mg_best')||'0');
    if (sc > best) { localStorage.setItem('mg_best', sc); best = sc; }
    this.scene.launch('UIScene', { win: true, score: sc, restartTo: 1 });
    this.scene.pause();
  }
};

PlayScene.prototype.update = function(time, delta) {
  const onLeft = this.leftDown();
  const onRight = this.rightDown();
  const running = this.runDown();

  const speed = running ? 220 : 140;
  if (onLeft) {
    this.player.setVelocityX(-speed);
  } else if (onRight) {
    this.player.setVelocityX(speed);
  } else {
    // keyboard
    if (this.cursors.left.isDown || this.keyA.isDown) this.player.setVelocityX(-speed);
    else if (this.cursors.right.isDown || this.keyD.isDown) this.player.setVelocityX(speed);
    else this.player.setVelocityX(0);
  }

  if ((this.cursors.up.isDown || this.keySpace.isDown) && this.player.body.touching.down) {
    this.player.setVelocityY(-420);
    this.sound.play('jump');
  }

  // score increases with time and passing obstacles
  let sc = this.registry.get('score') + Math.floor(delta/100);
  this.registry.set('score', sc);
  this.scoreText.setText('Score: ' + sc);
  this.bestText.setText('Best: ' + (localStorage.getItem('mg_best')||0));

  // move camera with player a bit
  // check remove obstacles beyond left
  this.obstacles.children.iterate(function(c){
    if (c.x < -50) c.destroy();
  });

  // basic gravity ground check to prevent falling out
  if (this.player.y > this.scale.height + 200) {
    this.hitObstacle();
  }
};

// UIScene: shows win/lose and restart
function UIScene() {
  Phaser.Scene.call(this, { key: 'UIScene' });
}
UIScene.prototype = Object.create(Phaser.Scene.prototype);
UIScene.prototype.constructor = UIScene;
UIScene.prototype.init = function(data) {
  this.data = data || {};
};
UIScene.prototype.create = function() {
  const w = this.scale.width, h = this.scale.height;
  const bg = this.add.rectangle(w/2, h/2, w*0.8, h*0.6, 0x000000, 0.7);
  const title = this.add.text(w/2, h/2 - 60, this.data.win ? 'Você Venceu!' : 'Você Perdeu', { fontSize:'28px', fill:'#fff' }).setOrigin(0.5);
  const txt = this.add.text(w/2, h/2 - 10, 'Pontuação: '+this.data.score, { fontSize:'20px', fill:'#fff' }).setOrigin(0.5);
  const btn = this.add.text(w/2, h/2 + 50, 'Jogar Novamente', { fontSize:'20px', fill:'#fff', backgroundColor:'#fff', color:'#000' }).setOrigin(0.5).setPadding(10).setInteractive();
  btn.on('pointerup', ()=> {
    this.scene.stop();
    this.scene.stop('PlayScene');
    this.scene.start('MenuScene');
  });
};
