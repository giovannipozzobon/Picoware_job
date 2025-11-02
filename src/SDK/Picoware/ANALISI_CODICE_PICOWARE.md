# ANALISI CODICE PICOWARE SDK

## Panoramica generale

Il progetto **Picoware** è un SDK per Raspberry Pi Pico progettato per creare applicazioni grafiche, giochi e interfacce utente su display LCD con controller **ST7789P** (320x320 pixel). Il sistema è strutturato in modo modulare con gestione avanzata della grafica, sprite ed entità di gioco.

---

## ARCHITETTURA DEL PROGETTO

### Struttura directory principale

```
src/
├── system/          # Componenti di sistema (input, storage, WiFi, LED)
│   └── drivers/     # Driver hardware (LCD, font, tastiera, SD card, PSRAM)
├── gui/             # Sistema grafico e UI
├── engine/          # Motore di gioco 2D/3D
└── applications/    # Applicazioni (giochi, screensaver, desktop)
    ├── desktop/
    ├── games/
    ├── library/
    ├── screensavers/
    └── wifi/
```

---

## GESTIONE GRAFICA

### 1. **Sistema LCD Driver** ([lcd.c](src/system/drivers/lcd.c), [lcd.h](src/system/drivers/lcd.h))

Il driver LCD gestisce il display ST7789P tramite interfaccia SPI.

#### Caratteristiche tecniche:
- **Display**: 320x320 pixel, RGB565 (16-bit color)
- **Interfaccia**: SPI a 75 MHz (massimo supportato: 62.5 MHz ufficiale)
- **Controller**: ST7789P
- **Frame memory**: 480 pixel di altezza (per scrolling verticale)

#### Funzionalità principali:
```c
void lcd_init(void)                    // Inizializza display
void lcd_blit(uint16_t *pixels, ...)   // Trasferisce pixel al display
void lcd_solid_rectangle(...)          // Disegna rettangoli pieni
void lcd_set_window(...)               // Imposta finestra di rendering
void lcd_putc(...)                     // Rendering carattere
void lcd_scroll_up/down()              // Scrolling verticale
```

#### Gestione colori:
- **RGB565**: 5 bit rosso, 6 bit verde, 5 bit blu
- Macro di conversione: `RGB(r, g, b)` converte RGB888 → RGB565
- Palette colori predefinite in [colors.hpp](src/system/colors.hpp):
  - `TFT_WHITE`, `TFT_BLACK`, `TFT_RED`, `TFT_GREEN`, `TFT_BLUE`, ecc.

---

### 2. **Classe Draw** ([draw.hpp](src/gui/draw.hpp), [draw.cpp](src/gui/draw.cpp))

La classe `Draw` è il layer di astrazione grafica principale, implementa **double buffering a 8 bit** con palette colori.

#### Architettura Double Buffering

```cpp
static uint8_t frontBuffer[320 * 320];   // Buffer visualizzato (8-bit)
static uint8_t backBuffer[320 * 320];    // Buffer di rendering (8-bit)
static uint16_t paletteBuffer[256];      // Palette RGB332 → RGB565
```

**Vantaggi dell'approccio 8-bit**:
- Risparmio memoria: 102.4 KB (8-bit) vs 204.8 KB (16-bit)
- Conversione automatica RGB332 → RGB565 tramite palette
- Swap ottimizzato con operazioni a 32-bit e DMA

#### Primitive grafiche

**Forme base**:
```cpp
void drawPixel(Vector position, uint16_t color)
void drawLine(Vector position, Vector size, uint16_t color)
void drawRect(Vector position, Vector size, uint16_t color)
void drawCircle(Vector position, int16_t r, uint16_t color)
void fillRect(Vector position, Vector size, uint16_t color)
void fillCircle(Vector position, int16_t r, uint16_t color)
void fillRoundRect(Vector position, Vector size, uint16_t color, int16_t radius)
void fillScreen(uint16_t color)
```

**Gestione buffer**:
```cpp
void swap(bool copyFrameBuffer, bool copyPalette)  // Swap buffer
void swapRegion(Vector position, Vector size)      // Swap parziale (ottimizzato UI)
void clearBuffer(uint8_t colorIndex)               // Pulisce back buffer
void clearBothBuffers(uint8_t colorIndex)          // Pulisce entrambi (anti-artifact)
```

#### Ottimizzazione rendering

1. **Conversione colore**:
   ```cpp
   uint8_t color332(uint16_t color)  // RGB565 → RGB332 (indice palette)
   uint16_t color565(uint8_t r, uint8_t g, uint8_t b)  // RGB888 → RGB565
   ```

2. **Swap ottimizzato**:
   - Operazioni a 32-bit per velocità
   - Trasferimenti DMA per SPI
   - Swap regionale per aggiornamenti UI parziali

3. **Clipping automatico**: tutte le primitive verificano i limiti dello schermo

---

### 3. **Sistema di Immagini** ([image.hpp](src/gui/image.hpp), [image.cpp](src/gui/image.cpp))

Gestisce sprite, bitmap e immagini BMP.

#### Classe Image

```cpp
class Image {
    uint16_t *buffer;           // Buffer RGB565 (per immagini 16-bit)
    uint8_t *data;              // Dati raw (8-bit o BMP)
    const uint8_t *progmemData; // Puntatore a dati in PROGMEM (flash)
    Vector size;                // Dimensioni (width, height)
    bool is8bit;                // Flag formato 8-bit
    bool ownsData;              // Gestione memoria (ownership)
};
```

#### Metodi di creazione immagini

1. **Da array di byte**:
   ```cpp
   bool fromByteArray(const uint8_t *data, Vector size, bool copy_data)
   ```
   - `copy_data=true`: copia in RAM (dati dinamici)
   - `copy_data=false`: riferimento diretto (PROGMEM)

2. **Da file BMP**:
   ```cpp
   bool fromPath(const char *path)  // Carica da SD card
   bool openImage(const char *path) // Apre BMP 16-bit
   bool createImageBuffer()         // Flip bottom-up → top-down
   ```

3. **Da PROGMEM** (flash):
   ```cpp
   bool fromProgmem(const uint8_t *progmem_ptr, Vector size)
   ```
   - Ottimizzato per sprite statici compilati nel firmware

4. **Da stringa** (per sprite testuali):
   ```cpp
   bool fromString(std::string data)  // Formato: '\n' = nuova riga, char = colore
   ```

#### ImageManager (Singleton)

Gestisce cache di immagini per evitare duplicazioni:

```cpp
ImageManager::getInstance().getImage(name, data, size, is8bit, isProgmem)
```

---

### 4. **Rendering Immagini/Sprite**

La classe `Draw` offre diverse modalità di rendering:

#### a) **Immagini con palette personalizzata**
```cpp
void image(
    Vector position,
    const uint8_t *bitmap,
    Vector size,
    const uint16_t *palette = nullptr,  // Palette opzionale
    bool imageCheck = true,             // Verifica boundary
    bool invert = false                 // Inversione pixel
)
```

#### b) **Immagini oggetto**
```cpp
void image(Vector position, Image *image, bool imageCheck = true)
```

#### c) **Bitmap 1-bit (monocromatico)**
```cpp
void imageBitmap(
    Vector position,
    const uint8_t *bitmap,  // Dati packed 1-bit (8 pixel/byte)
    Vector size,
    uint16_t color,
    bool invert = false
)
```
- Utilizzato per icone e glifi
- 1 bit = pixel on/off
- Memoria efficiente: 1/8 rispetto a 8-bit

#### d) **Immagini con colore singolo**
```cpp
void imageColor(
    Vector position,
    const uint8_t *bitmap,
    Vector size,
    uint16_t color,
    bool invert = false,
    uint8_t transparentColor = 0xFF  // Colore trasparente
)
```
- Converte immagine multi-colore in monocromatica
- Supporto trasparenza

---

## GESTIONE FONT

### 1. **Struttura Font** ([font.h](src/system/drivers/font.h))

I font sono definiti come array di byte con metadata:

```c
typedef struct {
    uint8_t width;      // Larghezza glifo (5 o 8 pixel)
    uint8_t glyphs[];   // Array glifi (10 byte/carattere)
} font_t;

#define GLYPH_HEIGHT 10  // Altezza fissa 10 pixel
```

### 2. **Font disponibili**

#### a) **Font 8x10** ([font-8x10.c](src/system/drivers/font-8x10.c))
- Basato su VT100/VT220 terminal font
- Dimensione: 8 pixel × 10 pixel
- Set completo ASCII (256 caratteri)
- Formato dati: ogni glifo = 10 byte (1 byte/riga, bit 7-0 = pixel)

Esempio glifo (carattere '0'):
```c
0b01111100,  // ┌─────┐ Riga 0
0b11000110,  // │███  │
0b11001110,  // │█ ██ │
0b11011110,  // │█ ███│
0b11110110,  // │████ │
0b11100110,  // │███  │
0b11000110,  // │█ ██ │
0b01111100,  // └─────┘ Riga 9
```

#### b) **Font 5x10** ([font-5x10.c](src/system/drivers/font-5x10.c))
- Versione compatta
- Dimensione: 5 pixel × 10 pixel
- Utilizzato per testo denso o numeri

### 3. **Rendering Testo**

#### Metodi della classe Draw

```cpp
void text(Vector position, const char text)                      // Singolo carattere
void text(Vector position, const char text, uint16_t color)      // Con colore
void text(Vector position, const char *text)                     // Stringa
void text(Vector position, const char *text, uint16_t color, int font=2)
```

#### Rendering carattere interno
```cpp
void renderChar(Vector position, char c, uint16_t color) {
    const font_t *currentFont = (font == 1) ? &font_8x10 : &font_5x10;
    int glyphIndex = c * GLYPH_HEIGHT;

    for (int row = 0; row < GLYPH_HEIGHT; row++) {
        uint8_t glyphRow = currentFont->glyphs[glyphIndex + row];

        if (currentFont->width == 8) {
            // Render 8-pixel font
            for (int col = 0; col < 8; col++) {
                if (glyphRow & (0x80 >> col))
                    drawPixel(Vector(position.x + col, position.y + row), color);
                else if (useBackgroundTextColor)
                    drawPixel(Vector(position.x + col, position.y + row), textBackground);
            }
        } else {
            // Render 5-pixel font (bit pattern: 0b000XXXXX)
            uint8_t bitMasks[5] = {0x10, 0x08, 0x04, 0x02, 0x01};
            for (int col = 0; col < 5; col++) {
                if (glyphRow & bitMasks[col])
                    drawPixel(Vector(position.x + col, position.y + row), color);
            }
        }
    }
}
```

#### Gestione colori testo
```cpp
void setForegroundTextColor(uint16_t color)  // Colore testo
void setBackgroundTextColor(uint16_t color)  // Colore sfondo
void setBackgroundTextStatus(bool status)    // Abilita/disabilita sfondo
```

---

## MOTORE DI GIOCO (ENGINE)

### 1. **Sistema di Entità** ([entity.hpp](src/engine/entity.hpp), [entity.cpp](src/engine/entity.cpp))

Implementa un sistema Entity-Component basato su callback.

#### Struttura Entity

```cpp
class Entity {
public:
    const char *name;
    Vector position, old_position;
    Vector direction, plane;        // Per rendering 3D
    Vector size;
    Image *sprite;                  // Sprite principale
    Image *sprite_left, *sprite_right;  // Sprite direzionali

    EntityType type;                // PLAYER, ENEMY, ICON, NPC, 3D_SPRITE
    EntityState state;              // IDLE, MOVING, ATTACKING, DEAD, ecc.

    // Proprietà gameplay
    float health, max_health;
    float speed, strength;
    float radius;                   // Collisione
    float level, xp;
    float attack_timer, elapsed_attack_timer;

    // 3D sprite support
    Sprite3D *sprite_3d;
    Sprite3DType sprite_3d_type;    // HUMANOID, TREE, HOUSE, PILLAR, CUSTOM
    float sprite_rotation, sprite_scale;

    // Callback functions
    void (*_start)(Entity*, Game*);
    void (*_stop)(Entity*, Game*);
    void (*_update)(Entity*, Game*);
    void (*_render)(Entity*, Draw*, Game*);
    void (*_collision)(Entity*, Entity*, Game*);
};
```

#### Creazione entità (esempio da SpaceInvaders):

```cpp
// Proiettile giocatore
Entity *bullet = new Entity(
    "PlayerBullet",
    ENTITY_ICON,
    Vector(x, y),
    Vector(2, 4),              // Dimensione sprite
    nullptr,                   // Nessun sprite (rendering custom)
    nullptr, nullptr,          // No sprite direzionali
    nullptr,                   // No start callback
    nullptr,                   // No stop callback
    [](Entity *bullet, Game *game) {  // Update callback
        bullet->position.y -= BULLET_SPEED;
        if (bullet->position.y < 0)
            bullet->is_active = false;
    },
    [](Entity *bullet, Draw *draw, Game *game) {  // Render callback
        draw->fillRect(bullet->position, bullet->size, TFT_YELLOW);
    }
);
```

#### Gestione sprite per entità

Due modalità:

1. **Sprite da oggetto Image**:
   ```cpp
   Entity(name, type, position, Image *sprite, ...)
   ```

2. **Sprite da byte array** (ottimizzato):
   ```cpp
   Entity(name, type, position, size, const uint8_t *sprite_data,
          const uint8_t *sprite_left_data,
          const uint8_t *sprite_right_data,
          ..., bool is_8bit_sprite, bool is_progmem_sprite)
   ```
   - `is_8bit_sprite`: usa palette 8-bit
   - `is_progmem_sprite`: dati in flash (risparmio RAM)

#### Rendering automatico sprite
```cpp
void Entity::render(Draw *draw, Game *game) {
    if (sprite != nullptr && is_visible) {
        draw->image(position, sprite, true);  // Rendering automatico
    }

    if (_render)  // Custom render callback
        _render(this, draw, game);
}
```

### 2. **Sprite 3D** ([sprite3d.hpp](src/engine/sprite3d.hpp))

Supporto per rendering pseudo-3D (raycasting-style):

```cpp
// Tipi di sprite 3D predefiniti
SPRITE_3D_HUMANOID  // Mesh umanoide
SPRITE_3D_TREE      // Albero
SPRITE_3D_HOUSE     // Casa
SPRITE_3D_PILLAR    // Colonna
SPRITE_3D_CUSTOM    // Custom mesh
```

#### Rendering 3D
```cpp
void Entity::render3DSprite(
    Draw *draw,
    Vector player_pos,      // Posizione camera
    Vector player_dir,      // Direzione vista
    Vector player_plane,    // Piano camera
    float view_height,
    Vector screen_size
)
```

Proiezione 3D → 2D tramite trasformazione prospettica.

### 3. **Classe Game** ([game.hpp](src/engine/game.hpp))

Container per gestione livelli e loop di gioco:

```cpp
class Game {
public:
    Draw *draw;
    Input *input_manager;
    Level *levels[MAX_LEVELS];  // Fino a 10 livelli
    Level *current_level;

    Vector camera;              // Posizione camera
    Vector pos;                 // Posizione giocatore
    CameraPerspective camera_perspective;  // FIRST_PERSON, TOP_DOWN, ecc.

    void start();
    void update();              // Loop principale
    void render();

    void level_add(Level *level);
    void level_switch(const char *name);
};
```

### 4. **Classe Level** ([level.hpp](src/engine/level.hpp))

Gestisce collezioni di entità:

```cpp
class Level {
    Entity *entities[MAX_ENTITIES];
    int entity_count;

    void entity_add(Entity *entity);
    void entity_remove(Entity *entity);
    Entity *getEntity(int index);

    void update(Game *game);    // Aggiorna tutte le entità
    void render(Draw *draw, Game *game);  // Renderizza tutte le entità
};
```

---

## GESTIONE SPRITE NEI GIOCHI

### Esempio: Space Invaders

#### 1. **Definizione costanti**
```cpp
#define INVADER_WIDTH 16
#define INVADER_HEIGHT 8
#define INVADER_SPACING_X 24
#define INVADER_SPACING_Y 16
```

#### 2. **Creazione nemici con sprite**
```cpp
// Invader con rendering custom
Entity *invader = new Entity(
    "Invader",
    ENTITY_ENEMY,
    Vector(x, y),
    Vector(INVADER_WIDTH, INVADER_HEIGHT),
    nullptr,  // Nessun sprite predefinito
    nullptr, nullptr,
    nullptr, nullptr,
    [](Entity *inv, Game *game) {  // Update
        // Logica movimento
    },
    [](Entity *inv, Draw *draw, Game *game) {  // Render
        // Disegno sprite invader pixel per pixel
        draw->fillRect(inv->position, Vector(4, 2), TFT_GREEN);
        draw->fillRect(Vector(inv->position.x+2, inv->position.y+2),
                      Vector(8, 4), TFT_GREEN);
        // ... definizione forma sprite
    }
);
```

### Esempio: Assets DOOM ([doom/assets.hpp](src/applications/games/doom/assets.hpp))

Sistema sprite più complesso:

```cpp
// Sprite font bitmap
#define bmp_font_width_pxs 192
#define bmp_font_height_pxs 48
#define CHAR_MAP " 0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ.,-_(){}[]#"

// Sprite armi
#define BMP_GUN_WIDTH 32
#define BMP_GUN_HEIGHT 32

// Sprite nemici (frame animazione)
#define BMP_IMP_WIDTH 32
#define BMP_IMP_HEIGHT 32
#define BMP_IMP_COUNT 5  // 5 frame animazione

// Sprite oggetti
#define BMP_ITEMS_WIDTH 16
#define BMP_ITEMS_HEIGHT 16
#define BMP_ITEMS_COUNT 2

// Gradient per shading
#define GRADIENT_WIDTH 2
#define GRADIENT_HEIGHT 8
#define GRADIENT_COUNT 8

// Array sprite in PROGMEM
extern const uint8_t bmp_gun[];
extern const uint8_t bmp_imp[BMP_IMP_COUNT][BMP_IMP_WIDTH * BMP_IMP_HEIGHT];
```

---

## OTTIMIZZAZIONI E BEST PRACTICES

### 1. **Gestione memoria**

#### Sprite in PROGMEM (flash)
```cpp
// Definizione in .cpp
PROGMEM const uint8_t mySprite[] = { /* dati */ };

// Uso
Entity *ent = new Entity(name, type, pos, size, mySprite,
                        nullptr, nullptr, nullptr, nullptr, nullptr, nullptr, nullptr,
                        true,   // is_8bit
                        true);  // is_progmem
```

#### ImageManager per riutilizzo
```cpp
Image *img = ImageManager::getInstance().getImage("player", spriteData, size, true, true);
// Richieste successive restituiscono la stessa istanza
```

### 2. **Rendering ottimizzato**

#### Swap parziale per UI
```cpp
// Aggiornamento solo area modificata
draw->swapRegion(Vector(x, y), Vector(width, height));
```

#### Clear buffer selettivo
```cpp
draw->clearBuffer(0);  // Solo back buffer (rendering normale)
draw->clearBothBuffers(0);  // Anti-artifact per cambi scena
```

### 3. **Sprite animation**

Cambio sprite direzionale automatico:
```cpp
entity->sprite_left = imgLeft;
entity->sprite_right = imgRight;
// Il motore scambia automaticamente in base a entity->direction
```

---

## COMPONENTI AGGIUNTIVI GUI

### Elementi UI ([gui/](src/gui/))

- **Alert** ([alert.cpp](src/gui/alert.cpp)): finestre di dialogo
- **Desktop** ([desktop.cpp](src/gui/desktop.cpp)): ambiente desktop
- **Keyboard** ([keyboard.cpp](src/gui/keyboard.cpp)): tastiera virtuale
- **List** ([list.cpp](src/gui/list.cpp)): liste scrollabili
- **Menu** ([menu.cpp](src/gui/menu.cpp)): menu navigabili
- **Scrollbar** ([scrollbar.cpp](src/gui/scrollbar.cpp)): barre di scorrimento
- **Textbox** ([textbox.cpp](src/gui/textbox.cpp)): input testuale
- **Toggle** ([toggle.cpp](src/gui/toggle.cpp)): switch on/off

Tutti utilizzano la classe `Draw` per rendering.

---

## CLASSE VECTOR

### Struttura base ([vector.hpp](src/gui/vector.hpp))

```cpp
class Vector {
public:
    float x, y;

    Vector(float x, float y);

    // Operazioni vettoriali
    Vector add(Vector a, Vector b);
    Vector sub(Vector a, Vector b);
    Vector mul(Vector a, Vector b);
    Vector div(Vector a, Vector b);

    // Overload operatori
    Vector operator+(const Vector &other);
    Vector operator-(const Vector &other);
    Vector operator*(const Vector &other);
    Vector operator/(const Vector &other);
    bool operator==(const Vector &other);
    bool operator!=(const Vector &other);
};
```

Utilizzata per:
- Posizioni entità
- Dimensioni sprite
- Velocità/direzione
- Coordinate schermo

---

## SISTEMA DI INPUT

### Input Manager ([input.hpp](src/system/input.hpp), [input.cpp](src/system/input.cpp))

Gestisce input da pulsanti hardware:

```cpp
// Costanti pulsanti (buttons.hpp)
#define BUTTON_UP       // D-pad su
#define BUTTON_DOWN     // D-pad giù
#define BUTTON_LEFT     // D-pad sinistra
#define BUTTON_RIGHT    // D-pad destra
#define BUTTON_A        // Pulsante A
#define BUTTON_B        // Pulsante B
#define BUTTON_CENTER   // Pulsante centrale
```

Utilizzo nei giochi:
```cpp
void player_update(Entity *entity, Game *game) {
    if (game->input == BUTTON_LEFT)
        entity->position.x -= entity->speed;
    if (game->input == BUTTON_RIGHT)
        entity->position.x += entity->speed;
    if (game->input == BUTTON_A)
        shootBullet(entity, game);
}
```

---

## VIEW MANAGER

### Gestione schermate ([view_manager.hpp](src/system/view_manager.hpp), [view_manager.cpp](src/system/view_manager.cpp))

Sistema per navigazione tra diverse "viste" (menu, giochi, applicazioni):

```cpp
class ViewManager {
    void add(View *view);              // Aggiunge vista
    void switchTo(const char *name);   // Cambia vista
    void run();                        // Loop principale
};
```

#### Struttura View ([view.hpp](src/system/view.hpp))

```cpp
class View {
    const char *name;
    void (*start)();    // Chiamata all'ingresso nella vista
    void (*stop)();     // Chiamata all'uscita dalla vista
    void (*loop)();     // Loop continuo mentre vista è attiva
};
```

Esempio di utilizzo ([Picoware.cpp](Picoware.cpp)):

```cpp
int main() {
    stdio_init_all();
    set_sys_clock_khz(200000, true);  // Overclock a 200 MHz

    ViewManager vm;
    vm.add(&desktopView);
    vm.switchTo("Desktop");

    while (true) {
        vm.run();
    }
}
```

---

## STORAGE E FILESYSTEM

### SD Card support ([sdcard.c](src/system/drivers/sdcard.c), [sdcard.h](src/system/drivers/sdcard.h))

Driver per interfaccia SD card via SPI.

### FAT32 filesystem ([fat32.c](src/system/drivers/fat32.c), [fat32.h](src/system/drivers/fat32.h))

Implementazione filesystem FAT32 per lettura/scrittura file.

### Storage Manager ([storage.cpp](src/system/storage.cpp), [storage.hpp](src/system/storage.hpp))

Layer di astrazione per operazioni filesystem:
- Caricamento/salvataggio configurazioni
- Caricamento asset (immagini BMP, sprite)
- Gestione save game

---

## SISTEMA WIFI

### WiFi support ([wifi.hpp](src/system/wifi.hpp), [wifi.cpp](src/system/wifi.cpp))

Gestione connessioni WiFi per:
- Applicazioni online
- Aggiornamenti OTA
- Sincronizzazione dati

### HTTP Client ([http.hpp](src/system/http.hpp), [http.cpp](src/system/http.cpp))

Client HTTP per API REST e fetch dati web.

---

## PSRAM SUPPORT

### PSRAM Driver ([rp2040-psram/psram_spi.c](src/system/drivers/rp2040-psram/psram_spi.c))

Supporto per memoria PSRAM esterna via SPI:
- Espansione RAM fino a 8 MB
- Utilizzata per buffer grafici aggiuntivi
- Storage temporaneo per asset di grandi dimensioni

### PSRAM Manager ([psram.hpp](src/system/psram.hpp), [psram.cpp](src/system/psram.cpp))

Gestione allocazione memoria PSRAM.

---

## LED CONTROL

### LED Manager ([led.cpp](src/system/led.cpp), [led.hpp](src/system/led.hpp))

Controllo LED di stato su board Raspberry Pi Pico:
- Indicatori stato sistema
- Feedback visivo operazioni
- Debug runtime

---

## APPLICAZIONI INCLUSE

### 1. **Desktop** ([applications/desktop/](src/applications/desktop/))

Ambiente desktop con:
- Launcher applicazioni
- Menu di sistema
- Gestione icone

### 2. **Giochi** ([applications/games/](src/applications/games/))

Collezione di giochi 2D/3D:

- **Space Invaders** ([spaceinvaders/](src/applications/games/spaceinvaders/)): clone classico arcade
- **Arkanoid** ([arkanoid/](src/applications/games/arkanoid/)): breakout-style game
- **Tetris** ([tetris/](src/applications/games/tetris/)): puzzle game classico
- **Pong** ([pong/](src/applications/games/pong/)): gioco ping-pong
- **Flappy Bird** ([flappybird/](src/applications/games/flappybird/)): endless runner
- **T-Rex Runner** ([trexrunner/](src/applications/games/trexrunner/)): Chrome dino game
- **Flight Assault** ([flightassault/](src/applications/games/flightassault/)): shoot'em up
- **Furious Birds** ([furiousbirds/](src/applications/games/furiousbirds/)): Angry Birds clone
- **Labyrinth** ([labyrinth/](src/applications/games/labyrinth/)): labirinto
- **DOOM** ([doom/](src/applications/games/doom/)): raycasting 3D FPS

### 3. **Screensaver** ([applications/screensavers/](src/applications/screensavers/))

Screensaver grafici:
- **Starfield** ([starfield/](src/applications/screensavers/starfield/)): campo stellare 3D
- **Cube** ([cube/](src/applications/screensavers/cube/)): cubo 3D rotante
- **Spiro** ([spiro/](src/applications/screensavers/spiro/)): spirografo

### 4. **Applicazioni WiFi** ([applications/wifi/](src/applications/wifi/))

- Scan reti WiFi
- Connessione access point
- Gestione credenziali

### 5. **Altre applicazioni** ([applications/applications/](src/applications/applications/))

- **GPS** ([GPS/](src/applications/applications/GPS/)): visualizzazione dati GPS
- **Weather** ([Weather/](src/applications/applications/Weather/)): meteo online
- **Flip Social** ([flip_social/](src/applications/applications/flip_social/)): social network

---

## CONFIGURAZIONE BUILD

### CMakeLists.txt

Il progetto utilizza CMake per build system:

```cmake
# Target principale
add_executable(Picoware Picoware.cpp)

# Link Pico SDK
pico_sdk_init()
target_link_libraries(Picoware pico_stdlib hardware_spi hardware_dma)

# Impostazioni display
target_compile_definitions(Picoware PRIVATE
    LCD_WIDTH=320
    LCD_HEIGHT=320
)
```

### Dipendenze

- **Pico SDK**: framework Raspberry Pi Pico
- **lwIP**: stack TCP/IP per networking
- **mbedTLS**: libreria crittografia per HTTPS

---

## PERFORMANCE E OTTIMIZZAZIONI

### 1. **Clock CPU**

Sistema overcloccato a **200 MHz** (default 125 MHz):
```cpp
set_sys_clock_khz(200000, true);
```

### 2. **SPI Display**

Velocità SPI **75 MHz** per trasferimenti rapidi:
```c
#define LCD_BAUDRATE (75000000)
```

### 3. **DMA Transfers**

Utilizzo DMA per trasferimenti SPI non-bloccanti:
```cpp
static int dma_channel = -1;
dma_channel = dma_claim_unused_channel(true);
// Trasferimenti buffer → display in background
```

### 4. **Memoria**

Ottimizzazioni memoria:
- Buffer 8-bit: 50% risparmio vs 16-bit
- Sprite PROGMEM: asset in flash invece di RAM
- ImageManager: cache condivisa immagini
- PSRAM: overflow per dati temporanei

### 5. **Rendering**

Tecniche ottimizzazione:
- Double buffering: elimina flickering
- Swap parziale: aggiorna solo regioni modificate
- Clipping: skip rendering fuori schermo
- Operazioni 32-bit: swap buffer 4x più veloce

---

## DEBUGGING E SVILUPPO

### 1. **Serial Output**

```cpp
stdio_init_all();  // Inizializza UART
printf("Debug info: %d\n", value);
```

### 2. **LED Status**

```cpp
led.set(true);   // LED on per debug
sleep_ms(500);
led.set(false);
```

### 3. **On-screen debug**

```cpp
draw->text(Vector(0, 0), "FPS: 60", TFT_WHITE);
draw->text(Vector(0, 10), buffer, TFT_YELLOW);
```

---

## ESEMPI DI UTILIZZO

### Esempio 1: Disegnare sprite personalizzato

```cpp
// Definizione sprite 16x16 (8-bit, PROGMEM)
PROGMEM const uint8_t playerSprite[16*16] = {
    0xFF, 0xFF, 0x00, 0x00, ... // Dati pixel RGB332
};

// Creazione entità
Image *img = ImageManager::getInstance().getImage(
    "player",
    playerSprite,
    Vector(16, 16),
    true,   // is_8bit
    true    // is_progmem
);

Entity *player = new Entity(
    "Player",
    ENTITY_PLAYER,
    Vector(160, 160),  // Centro schermo
    img
);

// Rendering
draw->image(player->position, player->sprite);
draw->swap();
```

### Esempio 2: Animazione sprite

```cpp
// Frame animazione
PROGMEM const uint8_t frame1[32*32] = { /* ... */ };
PROGMEM const uint8_t frame2[32*32] = { /* ... */ };
PROGMEM const uint8_t frame3[32*32] = { /* ... */ };

Image *frames[3] = {
    ImageManager::getInstance().getImageProgmem("f1", frame1, Vector(32, 32)),
    ImageManager::getInstance().getImageProgmem("f2", frame2, Vector(32, 32)),
    ImageManager::getInstance().getImageProgmem("f3", frame3, Vector(32, 32))
};

// Update loop
int currentFrame = 0;
uint32_t lastFrameTime = 0;

void update() {
    uint32_t now = to_ms_since_boot(get_absolute_time());
    if (now - lastFrameTime > 100) {  // 10 FPS animation
        currentFrame = (currentFrame + 1) % 3;
        lastFrameTime = now;
    }

    entity->sprite = frames[currentFrame];
}
```

### Esempio 3: Testo con font personalizzato

```cpp
Draw draw;

// Font 8x10 (default)
draw.setFont(1);
draw.text(Vector(10, 10), "Hello World", TFT_WHITE);

// Font 5x10 (compatto)
draw.setFont(2);
draw.text(Vector(10, 30), "Compact text", TFT_CYAN);

// Con background
draw.setBackgroundTextStatus(true);
draw.setBackgroundTextColor(TFT_BLUE);
draw.setForegroundTextColor(TFT_YELLOW);
draw.text(Vector(10, 50), "Highlighted");
```

### Esempio 4: Game loop completo

```cpp
Game *game = new Game(
    "MyGame",
    Vector(320, 320),  // Dimensione mondo
    &draw,
    &input,
    TFT_WHITE,         // Foreground
    TFT_BLACK          // Background
);

Level *level1 = new Level("Level 1", Vector(320, 320));

// Aggiungi entità
Entity *player = new Entity(/* ... */);
level1->entity_add(player);

game->level_add(level1);
game->level_switch(0);

// Main loop
while (true) {
    game->update();   // Update logica
    game->render();   // Rendering
    draw.swap();      // Swap buffer
}
```

---

## CONCLUSIONI

### Punti di forza:

1. **Architettura modulare**: separazione netta tra driver, grafica, engine e applicazioni
2. **Double buffering intelligente**: 8-bit con palette riduce uso RAM del 50%
3. **Flessibilità sprite**: supporto multi-formato (8/16-bit, PROGMEM, runtime)
4. **Sistema entità potente**: callback-based, supporto 2D/3D, gestione stato
5. **Font system**: rendering bitmap veloce, font multipli
6. **Ottimizzazione memoria**: PROGMEM per asset statici, ImageManager caching
7. **Estendibilità**: facile aggiungere nuovi giochi/applicazioni

### Caratteristiche tecniche chiave:

- **Display**: 320×320 RGB565 @ 75 MHz SPI
- **Buffer**: 8-bit (102.4 KB) con palette RGB332→RGB565
- **Font**: bitmap 5×10 e 8×10 pixel
- **Sprite**: 1-bit, 8-bit, 16-bit con trasparenza
- **Engine**: entity system con collision, health, AI callbacks
- **3D**: pseudo-3D raycasting per sprite volumetrici
- **CPU**: Raspberry Pi Pico @ 200 MHz (overclocked)
- **Memoria**: fino a 8 MB PSRAM esterna
- **Storage**: SD card FAT32
- **Networking**: WiFi + HTTP client

### Limitazioni:

1. **RAM limitata**: 264 KB (Pico), richiede gestione attenta memoria
2. **CPU single-core** (utilizzo limitato multicore)
3. **No GPU**: rendering completamente software
4. **Storage lento**: SD card via SPI (non SDIO)
5. **Display fisso**: ottimizzato per 320x320, non facilmente portabile

### Possibili miglioramenti:

1. **Sprite compression**: RLE o LZ per ridurre memoria
2. **Tile-based rendering**: per giochi 2D scrollabili
3. **Audio system**: sintesi/playback audio
4. **Particle system**: effetti particellari
5. **Physics engine**: collisioni avanzate, gravità
6. **Scripting**: Lua/JavaScript per logica gioco
7. **Level editor**: tool per creare livelli
8. **Multi-language**: internazionalizzazione font/UI

---

## RIFERIMENTI

### File chiave:

- **Entry point**: [Picoware.cpp](Picoware.cpp)
- **Grafica**: [src/gui/draw.hpp](src/gui/draw.hpp), [src/gui/draw.cpp](src/gui/draw.cpp)
- **Sprite**: [src/gui/image.hpp](src/gui/image.hpp), [src/gui/image.cpp](src/gui/image.cpp)
- **Font**: [src/system/drivers/font.h](src/system/drivers/font.h)
- **LCD**: [src/system/drivers/lcd.c](src/system/drivers/lcd.c)
- **Entity**: [src/engine/entity.hpp](src/engine/entity.hpp)
- **Game**: [src/engine/game.hpp](src/engine/game.hpp)

### Collegamenti esterni:

- **Raspberry Pi Pico SDK**: https://github.com/raspberrypi/pico-sdk
- **ST7789 Datasheet**: controller display LCD
- **Font originale**: VT100/VT220 terminal font

---

**Documento generato il**: 2025-11-02
**Versione SDK analizzata**: v1.5.1
**Autore analisi**: Claude (Anthropic)
