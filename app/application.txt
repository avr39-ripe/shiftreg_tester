#include <user_config.h>
#include <SmingCore/SmingCore.h>
#include <SPI.h>

const byte reg_in_latch = 4;
const byte reg_out_latch = 5;

const byte num_ch = 16;
const byte num_reg = num_ch / 8;

typedef union {
  uint32_t word;
  uint8_t bytes[4];
} TYPE_32BIT;

TYPE_32BIT out_reg;

TYPE_32BIT in_reg;

struct inPin {
  bool _mode;  //HIGH == pressed (1) or LOW == pressed (0)
    
  bool _lastState;
  bool _currentState;
  
  bool _debounced;
  bool _lastDebouncedState;
  bool _currentDebouncedState;
  unsigned long int _debounceTimerStartTime;
  unsigned int _debounceDelay;
  
  bool _pressed;
  bool _released;
  
  bool _changed;
  bool _justPressed;
  bool _justReleased;

  unsigned long int _currentTime; 
};

inPin inPins[num_ch];



Timer procTimer;

void debouncePin(byte pin);
void setupPin(byte pin, unsigned int debounceDelay, bool mode);
void loop();

void init()
{
	Serial.begin(SERIAL_BAUD_RATE); // 115200 by default
	Serial.systemDebugOutput(false); // Allow debug output to serial


	Serial.println("Sming start");

//  out_reg.word = 4294967295L;
//	out_reg.word = 43690;
	out_reg.word = 21845L;
  
  
  SPI.begin();
  
  unsigned long now = millis();
  
  for(int i = 0; i < num_ch; i++)
  {
    setupPin(i, 20, LOW);
  }
  

  pinMode(reg_in_latch, OUTPUT);
  digitalWrite(reg_in_latch, HIGH);
  
  pinMode(reg_out_latch, OUTPUT);
  digitalWrite(reg_out_latch, LOW);
  
  for(int i = 0; i < num_reg; i++)
  {
    SPI.transfer(0);
  }
  
  digitalWrite(reg_out_latch, HIGH);

  
  procTimer.initializeMs(100, loop).start();
}


void loop()
{
for(int c = 0; c < 2; c++) 
{
  unsigned long now;
  
  int byteIndex;
  int shiftIndex;
  
  
  
  digitalWrite(reg_in_latch, LOW);
  
  delayMicroseconds(100);
  
  digitalWrite(reg_in_latch, HIGH);
  
  digitalWrite(reg_out_latch, LOW);

  for(int i = 0; i < num_reg; i++)
  {
    in_reg.bytes[i] = SPI.transfer(out_reg.bytes[num_reg - 1 - i]);

    Serial.print("REG-IN"); Serial.print(i);Serial.print(" ");
    Serial.println(in_reg.bytes[i], BIN);
    Serial.print("REG-OUT"); Serial.print(i);Serial.print(" ");
    Serial.println(out_reg.bytes[i], BIN);
  }
  digitalWrite(reg_out_latch, HIGH);

for(int i = 0; i < num_ch; i++)
  {
    debouncePin(i);
     switch (i) {

     default:  
    if ( inPins[i]._changed && inPins[i]._pressed) {
           byteIndex = i / 8;
           shiftIndex = i % 8;
           out_reg.bytes[byteIndex] ^= (1 << shiftIndex);
         }
     }
}

}//for loop

}

void setupPin(byte pin, unsigned int debounceDelay, bool mode)
{
  inPins[pin]._mode = mode;
  inPins[pin]._lastState = 0;
    inPins[pin]._currentState = 0;
    
    inPins[pin]._debounced = 1;
    inPins[pin]._lastDebouncedState = 0;
    inPins[pin]._currentDebouncedState = 0;
    inPins[pin]._debounceTimerStartTime = 0;
    inPins[pin]._debounceDelay = debounceDelay;
    
    inPins[pin]._pressed = 0;
    inPins[pin]._released = 1;
  
    inPins[pin]._changed = 0;
    inPins[pin]._justPressed = 0;
    inPins[pin]._justReleased = 0;
     
}

void debouncePin(byte pin)
{
  byte byteId = pin / 8; 
  byte bitId = pin % 8;
  unsigned long int _currentTime = millis();
  
  inPins[pin]._currentState = in_reg.bytes[byteId] & (1 << bitId);
  
  if (inPins[pin]._currentState != inPins[pin]._lastState) {
    inPins[pin]._debounced = false;
    inPins[pin]._debounceTimerStartTime = _currentTime;
  } else if ((_currentTime - inPins[pin]._debounceTimerStartTime) > inPins[pin]._debounceDelay) {
    inPins[pin]._debounced = true;
  }
  
  if (inPins[pin]._debounced) {
    inPins[pin]._lastDebouncedState = inPins[pin]._currentDebouncedState;
    inPins[pin]._currentDebouncedState = inPins[pin]._currentState;
  }
  
  
  if (inPins[pin]._currentDebouncedState == inPins[pin]._mode) {
    inPins[pin]._pressed = true;
    inPins[pin]._released = false;
    inPins[pin]._justReleased = false;
  } else {
    inPins[pin]._pressed = false;
    inPins[pin]._released = true;
    inPins[pin]._justPressed = false;
  }
  
  
  if (inPins[pin]._lastDebouncedState != inPins[pin]._currentDebouncedState) {
    inPins[pin]._changed = true;
  } else {
    inPins[pin]._changed = false;
    inPins[pin]._justPressed = false;
    inPins[pin]._justReleased = false;
  }
  
  if (inPins[pin]._changed && inPins[pin]._pressed) {
    inPins[pin]._justPressed = true;
    inPins[pin]._justReleased = false;
  } else if(inPins[pin]._changed && inPins[pin]._released){
    inPins[pin]._justPressed = false;
    inPins[pin]._justReleased = true;
    
  } 
  inPins[pin]._lastState = inPins[pin]._currentState;
}