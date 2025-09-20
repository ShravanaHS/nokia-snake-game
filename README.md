# nokia-snake-game
This is a fun remake of the classic Nokia Snake game using Arduino Uno, an OLED display, and a joystick. Control the snake, eat food, and beat your high score! It has sound effects, adjustable speed, and saves your best score. Great for anyone curious about electronics and retro games.


# [CLICK HERE](https://shravanahs.github.io/nokia-snake-game/)
## 📽️ Demo Video
[![Watch the video](https://img.youtube.com/vi/0CRtBqZujK0/0.jpg)](https://youtu.be/0CRtBqZujK0?si=hq1VcGQwkqija8D)




# Source Code
### COPY AND PASTE IT IN ARDUINO IDE
```ARDUINO CODE
#include <U8g2lib.h>
#include <EEPROM.h>

// ---- OLED Setup ----
//U8G2_SH1106_128X64_NONAME_F_HW_I2C u8g2(U8G2_R0, /* reset=*/ U8X8_PIN_NONE);
U8G2_SH1106_128X64_NONAME_1_HW_I2C u8g2(U8G2_R0, /* reset=*/ U8X8_PIN_NONE);

// ---- Pin Definitions ----
#define JOY_X   A0
#define JOY_Y   A1
#define BTN     2
#define BUZZER  13

// --- Game Constants ---
#define SCREEN_W 128
#define SCREEN_H 64
#define GRID 4 // block size (pixels per cell)
#define CELLS_W (SCREEN_W/GRID)
#define CELLS_H (SCREEN_H/GRID)
#define INIT_LENGTH 3

enum GameState { MENU, PLAY, PAUSE, GAMEOVER, SHOW_SCORE, SETTINGS };
enum Joystick { NONE, UP, DOWN, LEFT, RIGHT, PRESS };

#define EEPROM_ADDR_HISCORE 0
#define MAX_SNAKE_LEN 100
//#define MAX_SNAKE_LEN (CELLS_W * CELLS_H)



// --- Global Variables ---
int snakeX[MAX_SNAKE_LEN], snakeY[MAX_SNAKE_LEN];
int snakeLen, dir, foodX, foodY, score, hiscore, speed;
bool alive, soundOn = true;

GameState gameState = MENU;
int menuIdx = 0; // main menu selection

const char* menuItems[] = {"Play", "High Score", "Sound", "Speed"};
const int menuCount = 4;
const int speeds[] = {200, 125, 75}; // ms per step (slow, norm, fast)
const char* speedLabels[] = {"Slow", "Normal", "Fast"};
int speedIdx = 1;

// --- Button Debounce ---
unsigned long lastBtnTime = 0;
const unsigned long btnDebounce = 150;

// --- Prototypes ---
void drawMenu();
void startGame();
void placeFood();
void moveSnake();
Joystick readJoy();
void playTone(int, int);
void drawHUD();
void drawSnake();
void drawFood();
void drawGameover();
void drawHighscore();
void drawPaused();

void setup() {
  pinMode(BTN, INPUT_PULLUP);
  pinMode(BUZZER, OUTPUT);
  u8g2.begin();
  u8g2.setFont(u8g2_font_5x7_tr);
  EEPROM.get(EEPROM_ADDR_HISCORE, hiscore);
  if (hiscore < 0 || hiscore > 999) hiscore = 0;
}

void loop() {
  static unsigned long lastMove = 0;
  Joystick j = readJoy();  // Read joystick and button once per loop

  switch (gameState) {
    case MENU:
      if (j == UP) {
        menuIdx = (menuIdx + menuCount - 1) % menuCount;
        playTone(1000,70);
      }
      else if (j == DOWN) {
        menuIdx = (menuIdx + 1) % menuCount;
        playTone(1000,70);
      }
      else if (j == PRESS) {
        playTone(1500,100);
        if (menuIdx == 0) {
          startGame();
        } else if (menuIdx == 1) {
          gameState = SHOW_SCORE;
        } else if (menuIdx == 2) {
          soundOn = !soundOn;
        } else if (menuIdx == 3) {
          speedIdx = (speedIdx + 1) % 3;
        }
      }
      drawMenu();
      break;

    case PLAY:
      if (j == PRESS) {
        gameState = PAUSE;
        playTone(700,110);
      }
      if (millis() - lastMove > speeds[speedIdx]) {
        lastMove = millis();
        // Update direction based on joystick tilt, ignore invalid reversals
        if (j == LEFT && dir != 3) dir = 1;
        else if (j == RIGHT && dir != 1) dir = 3;
        else if (j == UP && dir != 2) dir = 0;
        else if (j == DOWN && dir != 0) dir = 2;

        moveSnake();
      }
      u8g2.firstPage();
      do {
        drawHUD();
        drawSnake();
        drawFood();
      } while (u8g2.nextPage());
      break;

    case PAUSE:
      if (j == PRESS) {
        gameState = PLAY;
        playTone(800,60);
      }
      u8g2.firstPage();
      do {
        drawHUD();
        drawSnake();
        drawFood();
        drawPaused();
      } while (u8g2.nextPage());
      break;

    case GAMEOVER:
      if (j == PRESS) {
        gameState = MENU;
        playTone(1100,100);
      }
      u8g2.firstPage();
      do {
        drawGameover();
      } while (u8g2.nextPage());
      break;

    case SHOW_SCORE:
      if (j != NONE) {
        gameState = MENU;
        playTone(800,60);
      }
      u8g2.firstPage();
      do {
        drawHighscore();
      } while (u8g2.nextPage());
      break;

    default:
      break;
  }

  delay(20);  // Keep as global debounce/delay
}


// --- Support Functions ---

Joystick readJoy() {
  static bool btnPrev = HIGH;
  static unsigned long lastBtnTime = 0;
  static bool btnPressedFlag = false;

  int x = analogRead(JOY_X);
  int y = analogRead(JOY_Y);
  int b = digitalRead(BTN);
  Joystick evt = NONE;

  // Debounce button press
  if (b != btnPrev) {                          // Button state changed
    lastBtnTime = millis();                    // Reset timer
  }
  if ((millis() - lastBtnTime) > btnDebounce) {  // Stable for debounce time
    if (b == LOW && !btnPressedFlag) {        // Button pressed event
      evt = PRESS;
      btnPressedFlag = true;
    }
    else if (b == HIGH) {                      // Button released
      btnPressedFlag = false;
    }
  }
  btnPrev = b;

  // Joystick directions (center around 512)
  if (x < 400) evt = LEFT;
  else if (x > 600) evt = RIGHT;
  else if (y < 400) evt = UP;
  else if (y > 600) evt = DOWN;

  return evt;
}


void drawMenu() {
  u8g2.firstPage();
  do {
    u8g2.setFont(u8g2_font_6x12_tr);
    u8g2.drawStr(5,12, "Snake SHS");
    u8g2.setFont(u8g2_font_5x8_tr);
    for (int i=0;i<menuCount;i++) {
      if (menuIdx == i) u8g2.drawBox(1, 18+i*10, 125,9);
      u8g2.setDrawColor(menuIdx == i ? 0:1);
      String opt = menuItems[i];
      if (i==2) opt += (soundOn?" ON":" OFF");
      if (i==3) opt += String(" ") + speedLabels[speedIdx];
      u8g2.drawStr(8, 26+i*10, opt.c_str());
      u8g2.setDrawColor(1);
    }
    u8g2.setFont(u8g2_font_4x6_tr);
    u8g2.drawStr(90, 62, "Joystick: Up/Down | Btn=Select");
  } while (u8g2.nextPage());
}

void startGame() {
  snakeLen = INIT_LENGTH;
  dir = 3; // start right
  for(int i=0;i<snakeLen;i++) {
    snakeX[i]=CELLS_W/3-i;
    snakeY[i]=CELLS_H/2;
  }
  score = 0;
  alive = true;
  placeFood();
  gameState = PLAY;
}

void moveSnake() {
  // Move body
  for (int i=snakeLen-1; i>0; i--) {
    snakeX[i]=snakeX[i-1];
    snakeY[i]=snakeY[i-1];
  }
  // Move head
  if (dir==0) snakeY[0]--;
  else if (dir==1) snakeX[0]--;
  else if (dir==2) snakeY[0]++;
  else if (dir==3) snakeX[0]++;

  // Check wall collision
  if (snakeX[0]<0 || snakeX[0]>=CELLS_W || snakeY[0]<0 || snakeY[0]>=CELLS_H)
    { alive=false; gameState=GAMEOVER; playTone(200,300); }

  // Check self collision
  for (int i=1;i<snakeLen;i++)
    if (snakeX[0]==snakeX[i] && snakeY[0]==snakeY[i])
      { alive=false; gameState=GAMEOVER; playTone(200,300); }

  // Ate food
  if (snakeX[0] == foodX && snakeY[0] == foodY) {
    score++;
    if (snakeLen < MAX_SNAKE_LEN - 1) {
        // Initialize new tail block to same position as last segment
        snakeX[snakeLen] = snakeX[snakeLen - 1];
        snakeY[snakeLen] = snakeY[snakeLen - 1];
        snakeLen++;
    }
    if (soundOn) playTone(1200, 80);
    placeFood();
}

  if (!alive) {
    if (score>hiscore) {
      hiscore = score;
      EEPROM.put(EEPROM_ADDR_HISCORE, hiscore);
    }
  }
}

void placeFood() {
  bool conflict;
  do {
    foodX = random(CELLS_W);
    foodY = random(CELLS_H);
    conflict = false;
    for (int i = 0; i < snakeLen; i++) {
      if (snakeX[i] == foodX && snakeY[i] == foodY) {
        conflict = true;
        break;
      }
    }
  } while(conflict);
}

void playTone(int freq, int dur) {
  if (soundOn) tone(BUZZER, freq, dur);
}

void drawHUD() {
  // HiScore (top left), Live Score (top center), SHS (top right)
  u8g2.setFont(u8g2_font_4x6_tr);
  char buf[16];
  sprintf(buf,"HI:%d",hiscore);
  u8g2.drawStr(0,7,buf);

  sprintf(buf,"%d",score);
  int tw=u8g2.getStrWidth(buf);
  u8g2.drawStr((SCREEN_W-tw)/2,7,buf);

  u8g2.drawStr(SCREEN_W-19,7,"SHS");
}

void drawSnake() {
  // Draw snake body
  for (int i=1;i<snakeLen;i++) u8g2.drawBox(snakeX[i]*GRID, snakeY[i]*GRID,GRID,GRID);
  // Draw snake head (circular)
  u8g2.drawDisc(snakeX[0]*GRID+GRID/2, snakeY[0]*GRID+GRID/2, GRID/2, U8G2_DRAW_ALL);
}

void drawFood() {
  u8g2.drawBox(foodX*GRID,foodY*GRID,GRID,GRID);
}

void drawGameover() {
  u8g2.setFont(u8g2_font_6x12_tr);
  u8g2.drawStr((SCREEN_W-64)/2,24,"GAME OVER!");
  u8g2.setFont(u8g2_font_5x7_tr);
  char buf[31]; sprintf(buf,"Score: %d  High Score: %d",score, hiscore);
  u8g2.drawStr((SCREEN_W-u8g2.getStrWidth(buf))/2,36, buf);
  u8g2.setFont(u8g2_font_4x6_tr);
  u8g2.drawStr((SCREEN_W-56)/2,48,"Press Btn to Menu");
}

void drawPaused() {
  u8g2.setFont(u8g2_font_6x12_tr);
  u8g2.drawStr((SCREEN_W-40)/2,36,"PAUSED");
}

void drawHighscore() {
  u8g2.setFont(u8g2_font_6x12_tr);
  char buf[24]; sprintf(buf,"High Score:");
  u8g2.drawStr((SCREEN_W-u8g2.getStrWidth(buf))/2,28,buf);

  sprintf(buf,"%d",hiscore);
  u8g2.drawStr((SCREEN_W-u8g2.getStrWidth(buf))/2,44,buf);
  u8g2.setFont(u8g2_font_4x6_tr);
  u8g2.drawStr((SCREEN_W-56)/2,62,"Press Any Key to Menu");
}

```
