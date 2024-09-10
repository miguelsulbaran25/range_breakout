# range_breakout
bot trading  range breakout

//+------------------------------------------------------------------+
//|                                                    trakahner.mq5 |
//|                                  Copyright 2024, MetaQuotes Ltd. |
//|                                             https://www.mql5.com |
//+------------------------------------------------------------------+
#property version   "1.00"
//+------------------------------------------------------------------+
//| include                                                          |
//+------------------------------------------------------------------+
#include <Trade\Trade.mqh>
//+------------------------------------------------------------------+
//| inputs                                                           |
//+------------------------------------------------------------------+

input group    "==== entradas generales ===="
input long     InpMagicNumber    = 12345; //numero magico

enum LOT_MODE_ENUM{
      LOT_MODE_FIXED,                     //por lote
      LOT_MODE_MONEY,                     //por dinero
      LOT_MODE_PCT_ACCOUNT                //por % de dinero
};
input LOT_MODE_ENUM InpLotMode   = LOT_MODE_FIXED; //modo lote
input double   InpLots           = 0.1;   //lotaje / dinero / porcentaje
input int      InpStopLoss       = 100;   //stop loss  (0=off)
input bool     InpStopLossTrailing =true; //trailing stop 
input int      InpTakeProfit     = 300;   //take profit (0=off)

input group    "==== Break Even ===="
input bool     InpBreakEven = true;                 // Activar/Desactivar Break Even
input int      InpBreakEvenDistance = 50;           // Distancia de Break Even en pips
input int      InpBreakEvenActivationPoints = 200; // Cantidad de puntos de espacio para activar Break Even

input group    "==== entradas de rango ===="
input int      InpRangeStart     = 360;   //tiempo de inicio del rango en minutos
input int      InpRangeDuration  = 90;   //duracion de rango en minutos
input int      InpRangeClose     = 1200;  //tiempo de cierre del rango en minutos (-1 = off)

enum BREAKOUT_MODE_ENUM{
    ONE_SIGNAL,                           //una ruptura por rango
    TWO_SIGNALS                           //ruptura alta y baja
};
input BREAKOUT_MODE_ENUM InpBreakoutMode = ONE_SIGNAL; //modo ruptura
input group    "==== filtro dias de la semana ===="
input bool     InpMonday         = true;  //rango lunes
input bool     InpTuesday        = true;  //rango martes
input bool     InpWednesday      = true;  //rango miercoles
input bool     InpThursday       = true;  //rango jueves
input bool     InpFriday         = true;  //rango viernes

input group    "==== colores ===="
input color    InpColorRangeStart= clrBlue;        // Color de la línea de inicio del rango
input color    InpColorRangeHigh = clrBlueViolet;  // Color de la línea alta del rango
input color    InpColorRangeLow  = clrBlueViolet;  // Color de la línea baja del rango
input color    InpColorRangeEnd  = clrBlue;        // Color de la línea de fin del rango
input color    InpColorRangeClose= clrRed;         // Color de la línea de cierre del rango


//+------------------------------------------------------------------+
//| variables globales                                               |
//+------------------------------------------------------------------+
struct RANGE_STRUCT{
   datetime start_time;       // inicio del rango
   datetime end_time;         // fin del rango
   datetime close_time;       // cierre de tiempo
   double   high;             // alto del rango 
   double   low;              // bajo del rango
   bool     f_entry;          // marcar si estamos dentro del rango
   bool     f_high_breakout;  // marcar si se produjo una ruptura alta
   bool     f_low_breakout;   // marcar si se produjo una ruptura baja
   
   RANGE_STRUCT() : start_time(0), end_time(0), close_time(0),high(0),low(DBL_MAX),f_entry(false),f_high_breakout(false),f_low_breakout(false) {};
};

RANGE_STRUCT range;
MqlTick prevTick , lasTick;
CTrade trade;

bool BreakEven_Ready = false;

//+------------------------------------------------------------------+
//| Expert initialization function                                   |
//+------------------------------------------------------------------+
int OnInit()
  {
  
   //entradas de usuario
   if(!CheckInputs()){return INIT_PARAMETERS_INCORRECT;}
   
   //colocar numero magico
   trade.SetExpertMagicNumber(InpMagicNumber);
   
   // calcula el nuevo rango si hay nueva entrada 
   if(_UninitReason==REASON_PARAMETERS && CountOpenPositions()==0)
   {
      CalculateRange();
   }
   
   //crear objetos
   //DrawObjects();
   
   return(INIT_SUCCEEDED);
  }
//+------------------------------------------------------------------+
//| Expert deinitialization function                                 |
//+------------------------------------------------------------------+
void OnDeinit(const int reason)
  {
   // elimina los objetos
   ObjectsDeleteAll(NULL,"rango");
  }
//+------------------------------------------------------------------+
//| Expert tick function                                             |
//+------------------------------------------------------------------+
void OnTick()
  {
   // obtener el tick actual
   prevTick = lasTick;
   SymbolInfoTick(_Symbol,lasTick);
   
   // calculacion de rango
   if(lasTick.time >= range.start_time && lasTick.time < range.end_time){
      // establecer bandera
      range.f_entry = true;
      //nuevo alto
      if(lasTick.ask > range.high){
         range.high = lasTick.ask;
         DrawObjects();
      }
      //nuevo bajo
      if(lasTick.bid < range.low){
         range.low = lasTick.bid;
         DrawObjects();
      }
      
   }
   
   // cierre posiciones 
   if(InpRangeClose >=0 && lasTick.time >= range.close_time && CountOpenPositions() > 0){
      if(!ClosePositions()){return;}
   }
   
   //calcular el nuevo rango si....
   if (((InpRangeClose >=0 && lasTick.time>=range.close_time)                      // cierre tiempo
       || (range.f_high_breakout && range.f_low_breakout)                          // ruptura si es verdadero
       || (range.end_time==0)                                                      // rango no calculado
       || (range.end_time!=0 && lasTick.time>range.end_time && !range.f_entry))    // se calculo el rango pero no hay informacion de marca
       && CountOpenPositions ()==0){
       
       CalculateRange();
   }
   
   //verificar los rangos
   CheckBreakouts();
   
   // actualizar stop loss
   UpdateStopLoss();
   
   // break even
   UpdateBreakEven();
}

// verifica entradas de usuario
bool CheckInputs(){
   if(InpMagicNumber<=0)
   {
      Alert("Numero magico <= 0");
      return false;
   }
   
   if(InpLotMode == LOT_MODE_FIXED && (InpLots <= 0 || InpLots > 1000))
   {
      Alert("lotaje <= 0 o > 10");
      return false;
   }
   
   if(InpLotMode == LOT_MODE_MONEY && (InpLots <= 0 || InpLots > 1000))
   {
      Alert("dinero <= 0 o > 1000");
      return false;
   }
   
   if(InpLotMode == LOT_MODE_PCT_ACCOUNT && (InpLots <= 0 || InpLots > 100))
   {
      Alert("porcentaje <= 0 o > 100%");
      return false;
   }
   
   if((InpLotMode == LOT_MODE_MONEY || InpLotMode == LOT_MODE_PCT_ACCOUNT) && InpStopLoss == 0)
   {
      Alert("selecciona un modo de lote necesito un stop loss");
      return false;
   }
   
   if(InpStopLoss < 0 || InpLots > 1000)
   {
      Alert("Stop loss < 0 o > 1000");
      return false;
   }
   
   if(InpTakeProfit < 0 || InpTakeProfit > 1000)
   {
      Alert("take profit < 0 o > 1000");
      return false;
   }
   
   if(InpRangeClose < 0 && InpStopLoss == 0)
   {
      Alert("cierre de tiempo y stop loss off");
      return false;
   }
   
   
   if(InpRangeStart < 0 || InpRangeStart >= 1440)
   {
      Alert("rango inicio < 0 o >= 1440");
      return false;
   }
   
   if(InpRangeDuration <= 0 || InpRangeDuration >= 1440)
   {
      Alert("duracion de rango <= 0 o >= 1440");
      return false;
   }
   
   if(InpRangeClose >=1440  || (InpRangeStart+InpRangeDuration)%1440 == InpRangeClose)
   {
      Alert("tiempo de cierre >= 1440 o fin de tiempo == cierre tiempo");
      return false;
   }
   
   if(InpMagicNumber+InpMonday+InpTuesday+InpWednesday+InpThursday+InpFriday==0)
   {
      Alert("rango es prohibido todos los dias de la semana");
      return false;
   }
   
   return true;
}

// caculamos el nuevo rango
void CalculateRange(){

   // resetear las variables del rango 
   range.start_time = 0;
   range.end_time  = 0;
   range.close_time= 0;
   range.high = 0.0;
   range.low = DBL_MAX;
   range.f_entry = false;
   range.f_high_breakout = false;
   range.f_low_breakout = false;
   
   // calculamos el tiemp ode rango
   int time_cycle = 86400;
   range.start_time = (lasTick.time - (lasTick.time % time_cycle)) + InpRangeStart * 60;
   for (int i= 0; i<8; i ++){
      MqlDateTime tmp;
      TimeToStruct(range.start_time,tmp);
      int dow = tmp.day_of_week;
      if(lasTick.time>=range.start_time || dow==6 || dow == 0 || (dow==1 && !InpMonday) || (dow==2 && !InpTuesday)
         || (dow==3 && !InpWednesday)|| (dow==4 && !InpThursday)|| (dow==5 && !InpFriday)){
         range.start_time += time_cycle;
      }
   }
   
   // calcula el fin de tiempo de rango 
   range.end_time = range.start_time + InpRangeDuration * 60;
   for(int i = 0; i<2; i++){
      MqlDateTime tmp;
      TimeToStruct(range.end_time,tmp);
      int dow = tmp.day_of_week;
      if(dow==6 || dow ==0){
         range.end_time += time_cycle;
      }
   }
   
   // calcula el cierre de rango 
   if(InpRangeClose>=0){
      range.close_time = (range.end_time - (range.end_time % time_cycle)) + InpRangeClose *60;
      for(int i= 0; i<3; i++){
         MqlDateTime tmp;
         TimeToStruct(range.close_time,tmp);
         int dow = tmp.day_of_week;
         if(range.close_time<=range.end_time || dow==6 || dow==0){
            range.close_time+= time_cycle;
         }
      }
   }
   // creacion de objetos
   DrawObjects();
}

// CONTEO DE TODAS LAS POSICIONES
int CountOpenPositions(){

   int counter = 0;
   
   for(int i = PositionsTotal()-1; i >=0; i--){
      ulong ticket = PositionGetTicket(i);
      if(ticket<=0){Print("fallo en obtener la posicion del ticket"); return -1;}
      if(!PositionSelectByTicket(ticket)){Print("fallo en seleccionar la posicion de by ticket"); return -1;}
      ulong magicnumber;
      if(!PositionGetInteger(POSITION_MAGIC,magicnumber)){Print("fallo en obtener la posicion de magicnumber"); return -1;}
      if(InpMagicNumber==magicnumber){counter++;}
   }
   
   if(!counter) BreakEven_Ready = false;

   return counter;
}

// tome desgloses
void CheckBreakouts(){

   // comprobar si estamos detras del rango final
   if(lasTick.time >= range.end_time && range.end_time > 0 && range.f_entry){
   
      //comprobar si hay ruptura al alza
      if(!range.f_high_breakout && lasTick.ask >= range.high){
         range.f_high_breakout = true;
         if(InpBreakoutMode==ONE_SIGNAL){range.f_low_breakout = true;}
         
         // calcula el stop loss y el trake profit
         double sl =   InpStopLoss == 0 ? 0 : NormalizeDouble (lasTick.bid - ((range.high-range.low) * InpStopLoss * 0.01),_Digits);
         double tp =   InpTakeProfit == 0 ? 0 : NormalizeDouble (lasTick.bid + ((range.high-range.low) * InpTakeProfit * 0.01),_Digits);
         
         // calcula el lotaje
         double lots;
         if(!CalculateLots(lasTick.bid-sl,lots)){return;}
         
         // abrir operacion de compra 
         trade.PositionOpen(_Symbol,ORDER_TYPE_BUY,lots,lasTick.ask,sl,tp,"rompe rango EA");
      }
      
      //comprobar si hay ruptura a la baja 
      if(!range.f_low_breakout && lasTick.bid <= range.low){
         range.f_low_breakout = true;
         if(InpBreakoutMode==ONE_SIGNAL){range.f_high_breakout = true;}
         
          // calcula el stop loss y el trake profit
         double sl =    InpStopLoss == 0 ? 0 : NormalizeDouble (lasTick.ask + ((range.high-range.low) * InpStopLoss * 0.01),_Digits);
         double tp =    InpTakeProfit == 0 ? 0 : NormalizeDouble (lasTick.ask - ((range.high-range.low) * InpTakeProfit * 0.01),_Digits);
         
          // calcula el lotaje
         double lots;
         if(!CalculateLots(sl-lasTick.bid,lots)){return;}
         
         // abrir operacion de venta
         trade.PositionOpen(_Symbol,ORDER_TYPE_SELL,lots,lasTick.bid,sl,tp,"rompe rango EA");
      }
      
   }
}

// calculamos los lotes
bool CalculateLots(double slDistance, double &lots){
   
   lots = 0.0;
   if(InpLotMode == LOT_MODE_FIXED){
      lots = InpLots;
   }
   else{
       double tickSize    = SymbolInfoDouble(_Symbol,SYMBOL_TRADE_TICK_SIZE);
       double tickValue   = SymbolInfoDouble(_Symbol,SYMBOL_TRADE_TICK_VALUE);
       double VolumenStep = SymbolInfoDouble(_Symbol,SYMBOL_VOLUME_STEP);
   
      double  riskMoney   = InpLotMode==LOT_MODE_MONEY ? InpLots : AccountInfoDouble(ACCOUNT_EQUITY) * InpLots * 0.01;
      double  moneyVolumeStp = (slDistance / tickSize) * tickValue * VolumenStep;
      
      lots = MathFloor(riskMoney/moneyVolumeStp) * VolumenStep;
   }
   
   // verifica lotes calculados
   if(!CheckLots(lots)){return false;}

   return true;
}

// verifica los lotes maximo y minimos
bool CheckLots(double &lots){
   
   double min = SymbolInfoDouble(_Symbol,SYMBOL_VOLUME_MIN);
   double max = SymbolInfoDouble(_Symbol,SYMBOL_VOLUME_MAX);
   double step = SymbolInfoDouble(_Symbol,SYMBOL_VOLUME_STEP);
   
   if(lots<min){
      Print("el tamaño del lote se establecera en el volumen minimo permitido");
      lots = min;
      return true;
   }
   if(lots>max){
      Print("tamaño del lote mayor que el volumen permitido. lote:",lots,"max:",max);
      return false;
   }
   
   lots = (int)MathFloor(lots/step) * step;
   
   return true;
}


// actualiza sl a break even
void UpdateBreakEven(){
   
   // Devuelve si el stop loss no está establecido o el trailing stop para el stop loss está deshabilitado
   if(!InpBreakEven || CountOpenPositions() == 0){return;}
   
   if(InpBreakEven && BreakEven_Ready){return;}
   
   // Iterar a través de posiciones abiertas
   for(int i = PositionsTotal()-1; i >=0; i--){
      ulong ticket = PositionGetTicket(i);
      if(ticket <= 0){Print("Failed to get position ticket"); return;}
      if(!PositionSelectByTicket(ticket)){Print("Failed to select position by ticket"); return;}
      ulong magicnumber;
      if(!PositionGetInteger(POSITION_MAGIC, magicnumber)){Print("Failed to get position magic number"); return;}
      if(InpMagicNumber == magicnumber){
         
         // Obtener tipo de posición, detener pérdidas y obtener ganancias
         long type;
         if(!PositionGetInteger(POSITION_TYPE, type)){Print("Failed to get position type"); return;}
         double currSL, currTP, currPriceOpen;
         if(!PositionGetDouble(POSITION_SL, currSL)){Print("Failed to get position stop loss"); return;}
         if(!PositionGetDouble(POSITION_TP, currTP)){Print("Failed to get position take profit"); return;}
         if(!PositionGetDouble(POSITION_PRICE_OPEN, currPriceOpen)){Print("Failed to get position price open"); return;}
         
         // Calcular el nivel de equilibrio
         double breakEvenLevel;
         if(type == POSITION_TYPE_BUY){
            breakEvenLevel = NormalizeDouble(currPriceOpen + InpBreakEvenDistance * _Point, _Digits);
         } else {
            breakEvenLevel = NormalizeDouble(currPriceOpen - InpBreakEvenDistance * _Point, _Digits);
         }
         
         // Comprobar si el precio actual está por encima del nivel de equilibrio para posiciones largas
         // o debajo para posiciones cortas
         double currPrice = type == POSITION_TYPE_BUY ? lasTick.ask : lasTick.bid;
         double activationPrice = type == POSITION_TYPE_BUY ? (currPriceOpen + InpBreakEvenActivationPoints * _Point) :
                                                              (currPriceOpen - InpBreakEvenActivationPoints * _Point);
         
         if(((type == POSITION_TYPE_BUY && currPrice >= activationPrice) || (type == POSITION_TYPE_SELL && currPrice <= activationPrice))
            && currSL != breakEvenLevel){
            //Actualiza el stop loss al nivel de equilibrio
            if(!trade.PositionModify(ticket, breakEvenLevel, currTP)){
               Print("Failed to modify position", (string)ticket, "SL:", (string)currSL, "BE:", (string)breakEvenLevel, "TP:", (string)currTP);
               return;
            } else {
               BreakEven_Ready = true;
            }
         }
      }
   }
}

// actualizar stop loss
void UpdateStopLoss(){
   
   if(CountOpenPositions() == 0){return;}
   // return si el stop loss no esta fijado
   if(InpStopLoss == 0 || !InpStopLossTrailing){return;}
   
   if(InpBreakEven && !BreakEven_Ready){return;}
    
   // recorrer posiciones abiertas
   for(int i = PositionsTotal()-1; i >=0; i--){
      ulong ticket = PositionGetTicket(i);
      if(ticket<=0){Print("fallo en obtener la posicion del ticket"); return;}
      if(!PositionSelectByTicket(ticket)){Print("fallo en seleccionar la posicion de by ticket"); return;}
      ulong magicnumber;
      if(!PositionGetInteger(POSITION_MAGIC,magicnumber)){Print("fallo en obtener la posicion de magicnumber"); return;}
      if(InpMagicNumber==magicnumber){
         
         // obtener 
         long type;
         if(!PositionGetInteger(POSITION_TYPE,type)){Print("fallo en obtener la posicion de tipo"); return;}
         // obtener sl y tp
         double currSL, currTP;
         if(!PositionGetDouble(POSITION_SL,currSL)){Print("fallo en obtener la posicion de stop loss"); return;}
         if(!PositionGetDouble(POSITION_TP,currTP)){Print("fallo en obtener la posicion de take profit"); return;}
         
         // calcula el nuevo stop loss
         double currPrice = type==POSITION_TYPE_BUY ? lasTick.bid : lasTick.ask;
         int n            = type==POSITION_TYPE_BUY ? 1 : -1;
         double newSL     = InpBreakEven ? 
                                NormalizeDouble(currPrice - ((InpBreakEvenActivationPoints * _Point) * n),_Digits)
                              : NormalizeDouble(currPrice - ((range.high-range.low) * InpStopLoss * 0.01 * n),_Digits); // de dónde salio ese 0.01?

         // verificamos el nuevo stop loss con el precio existente
         if((newSL*n) < (currSL*n) || NormalizeDouble(MathAbs(newSL-currSL),_Digits)<_Point){
            Print("no hay nuevo stop loss");
            continue;
         }
         
         // verificamos el nivel de stop
         long level = SymbolInfoInteger(_Symbol, SYMBOL_TRADE_STOPS_LEVEL);
         if(level!=0 && MathAbs(currPrice-newSL)<=level*_Point){
            Print("nuevo stop los en el nivel");
            continue;
         }
         
         // modificamos la position con el nuevo stop loss
         if(!trade.PositionModify(ticket,newSL,currTP)){
            Print("fallo en modificar la posicion",(string)ticket,"currSL:",(string)currSL,"newSL:",
            (string)newSL,"currTP:",(string)currTP);
            
            return;
         }
         
      }
   }

}

// cierre de todas las posiciones
bool ClosePositions(){
   
   for(int i = CountOpenPositions()-1; i>=0; i--){
      //if(total != PositionsTotal()){total=PositionsTotal(); i= total;continue;} // CountOpenPositions ya te trae las posiciones de este EA
      ulong ticket = PositionGetTicket(i); // seleciona positiones
      if(ticket <=0 ){Print("fallo en obtener la posicion"); return false;}
      if(!PositionSelectByTicket(ticket)){Print("fallo en seleccionar la posicion"); return false;}
      long magicnumber; 
      if(!PositionGetInteger(POSITION_MAGIC,magicnumber)){Print("fallo en obtener el numero magico"); return false;}
      if(magicnumber == InpMagicNumber){
         trade.PositionClose(ticket);
         BreakEven_Ready = false;
         
         if(trade.ResultRetcode() != TRADE_RETCODE_DONE){
            Print("fallo en cerrar la posicion result: "+(string)trade.ResultRetcode()+":"+trade.ResultRetcodeDescription());
            return false;
         }
      }
   }
   
   return true;
}

// objetos
void DrawObjects(){
    // Empezar rango
   if(range.start_time>0){
      ObjectCreate(NULL,"empezar rango",OBJ_VLINE,0,range.start_time,0);
      ObjectSetString(NULL,"empezar rango",OBJPROP_TOOLTIP,"empezar rango \n"+TimeToString(range.start_time,TIME_DATE|TIME_MINUTES));
      ObjectSetInteger(NULL,"empezar rango",OBJPROP_COLOR,InpColorRangeStart);
      ObjectSetInteger(NULL,"empezar rango",OBJPROP_WIDTH,2);
      ObjectSetInteger(NULL,"empezar rango",OBJPROP_BACK,true);
   }
   
   // Fin rango
   if(range.end_time>0){
      ObjectCreate(NULL,"fin rango",OBJ_VLINE,0,range.end_time,0);
      ObjectSetString(NULL,"fin rango",OBJPROP_TOOLTIP,"fin del rango \n"+TimeToString(range.end_time,TIME_DATE|TIME_MINUTES));
      ObjectSetInteger(NULL,"fin rango",OBJPROP_COLOR,InpColorRangeEnd);
      ObjectSetInteger(NULL,"fin rango",OBJPROP_WIDTH,2);
      ObjectSetInteger(NULL,"fin rango",OBJPROP_BACK,true);
   }
   
   // Cierre de tiempo 
   if(range.close_time>0){
      ObjectCreate(NULL,"cierre rango",OBJ_VLINE,0,range.close_time,0);
      ObjectSetString(NULL,"cierre rango",OBJPROP_TOOLTIP,"cierre del rango \n"+TimeToString(range.close_time,TIME_DATE|TIME_MINUTES));
      ObjectSetInteger(NULL,"cierre rango",OBJPROP_COLOR,InpColorRangeClose);
      ObjectSetInteger(NULL,"cierre rango",OBJPROP_WIDTH,2);
      ObjectSetInteger(NULL,"cierre rango",OBJPROP_BACK,true);
   }
   
   // Alto
   ObjectsDeleteAll(NULL,"rango alto");
      if(range.high>0){
      ObjectCreate(NULL,"rango alto",OBJ_TREND,0,range.start_time,range.high,range.end_time,range.high);
      ObjectSetString(NULL,"rango alto",OBJPROP_TOOLTIP,"rango alto \n"+DoubleToString(range.high,_Digits));
      ObjectSetInteger(NULL,"rango alto",OBJPROP_COLOR,InpColorRangeLow);
      ObjectSetInteger(NULL,"rango alto",OBJPROP_WIDTH,2);
      ObjectSetInteger(NULL,"rango alto",OBJPROP_BACK,true);
      ObjectSetInteger(NULL,"rango alto",OBJPROP_STYLE,STYLE_DOT);
      
      ObjectCreate(NULL,"rango alto guia",OBJ_TREND,0,range.end_time,range.high, InpRangeClose >= 0 ? range.close_time : INT_MAX,range.high);
      ObjectSetString(NULL,"rango alto guia",OBJPROP_TOOLTIP,"rango alto guia\n"+DoubleToString(range.high,_Digits));
      ObjectSetInteger(NULL,"rango alto guia",OBJPROP_COLOR,InpColorRangeLow);
      ObjectSetInteger(NULL,"rango alto guia",OBJPROP_BACK,true);
      ObjectSetInteger(NULL,"rango alto guia",OBJPROP_STYLE,STYLE_DOT);
   }
   
   // Bajo
   ObjectsDeleteAll(NULL,"rango bajo");
      if(range.low<DBL_MAX){
      ObjectCreate(NULL,"rango bajo",OBJ_TREND,0,range.start_time,range.low,range.end_time,range.low);
      ObjectSetString(NULL,"rango bajo",OBJPROP_TOOLTIP,"rango bajo \n"+DoubleToString(range.low,_Digits));
      ObjectSetInteger(NULL,"rango bajo",OBJPROP_COLOR,InpColorRangeLow);
      ObjectSetInteger(NULL,"rango bajo",OBJPROP_WIDTH,2);
      ObjectSetInteger(NULL,"rango bajo",OBJPROP_BACK,true);
      ObjectSetInteger(NULL,"rango bajo",OBJPROP_STYLE,STYLE_DOT);
      
      ObjectCreate(NULL,"rango bajo guia",OBJ_TREND,0,range.end_time,range.low, InpRangeClose >= 0 ? range.close_time : INT_MAX,range.low);
      ObjectSetString(NULL,"rango bajo guia",OBJPROP_TOOLTIP,"rango bajo guia \n"+DoubleToString(range.low,_Digits));
      ObjectSetInteger(NULL,"rango bajo guia",OBJPROP_COLOR,InpColorRangeLow);
      ObjectSetInteger(NULL,"rango bajo guia",OBJPROP_BACK,true);
      ObjectSetInteger(NULL,"rango bajo guia",OBJPROP_STYLE,STYLE_DOT);
   }

   // refrescamos el grafico
   ChartRedraw();
}
