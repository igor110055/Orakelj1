# Orakelj1

Recimo, da izdelujemo Dapp in hočemo uporabnikom aplikacije omogočiti, da dvignejo Eth v določeni količini USD.
Da zadostimo tej zahtevi, mora naša pametna pogodba poznati vrednost Eth v USD. 

JavaScript aplikacija lahko preprosto prenese tako vrsto informacije z Binance public API (ali s kakšne druge storitve). Ker pametna pogodba nima 
direktnega dostopa do kakršnihkoli informacij z zunanjega sveta, se za dostop do podatkov zanaša na orakelj.

# Postavljanje ogrodja

V terminalu se postavimo v mapo našega projekta. Nato naredimo mapo EthPriceOracle in se postavimo v njo. 

Zaženemo ukaz:

```
npm init -y
```

Nato pa uvozimo naslednje pakete: truffle, openzeppelin-solidity, loom-js, loom-truffle-provider, bn.js in axios.

(pri uvozu loom-js je morda potrebno naložiti verzijo loom-js@1.75.2)

Zdaj, ko imamo vse pripravljeno, naredimo dve mapi oracle in caller ter v njih zaženemo truffle.
To storimo tako, da zaženemo spodnja ukaza:

```
mkdir oracle && cd oracle && npx truffle init && cd ..

mkdir caller && cd caller && npx truffle init && cd ..
```

Naša mapa naj bi imela spodnjo obliko:
```
.
├── caller
│   ├── contracts
│   ├── migrations
│   ├── test
│   └── truffle-config.js
├── oracle
│   ├── contracts
│   ├── migrations
│   ├── test
│   └── truffle-config.js
└── package.json

```

# Klicanje drugih pogodb

Za boljše razumevanje bomo začeli s pisanjem caller.sol pogodbe. 
Ena od stvari, ki jih bo pogodba počela je, da bo v interakciji z orakljem. 
Zato, da bo to lahko počela, ji moramo podati naslednji informaciji:

- Naslov pametne pogodbe oraklja
- Podpis funkcije, ki jo želimo klicati

Najlažji pristop bi bil kar ročno vnašanje teh podatkov. Odgovor na zakaj to ne bi bilo pametno je, 
da ko je pogodba enkrat deployana, je ni mogoče spreminjati. Če malo premislimo, hitro najdemo
kar nekaj primerov, v katerih bi želeli posodobiti naslov oraklja. Da problem rešimo, napišemo 
funkcijo, ki shrani naslov pogodbe oraklja v spremenljivki, nato pa sproži pametno pogodbo oraklja,
tako da lahko pogodba kadarkoli kliče njene funkcije.

Začnimo s pisanjem pogodbe CallerContract.sol v mapi caller. Zgornji razmislek lahko implementiramo 
v pogodbo tako, da vanjo prilepimo spodnjo kodo:

``` Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
contract CallerContract {
    address private oracleAddress;
    function setOracleInstanceAddress (address _oracleInstanceAddress) public {
      oracleAddress = _oracleInstanceAddress;
    }
}

```
# Klicanje funkcij z drugih pogodb

Zdaj si poglejmo, kako lahko kličemo funkcijo z druge pogodbe.

Da je lahko caller pogodba v interakciji z oracle pogodbo, moramo definirati interface.
Interface je nekaj, kar je podobno pogodbi, le da ne more imeti funkcij implementiranih.
Interface med drugim ne more definirati spremenljivk, konstruktorjev, prav tako pa ne more 
dedovati z drugih pogodb. Najlažje si Interface predstavljamo kot nek ABI. 
Ker dovoljujejo drugim pogodbam, da so v interakciji ena z drugo, morajo biti funkcije tipa external.

Za boljšo predstavo si lahko pogledamo naslednji primer. Recimo, da obstaja pogodba, imenovana 
FastFood, ki izgleda takole:

``` Solidity
pragma solidity 0.5.0;

contract FastFood {
  function makeSandwich(string calldata _fillingA, string calldata _fillingB) external {
    //Make the sandwich
  }
}
```

Ta preprosta pogodba implementira funkcijo, ki naredi sendvič. Če poznamo naslov FastFood pogodbe in 
podpis funkcije makeSandwich, jo lahko kličemo.

Opomba: Podpis funkcije vsebuje njeno ime, seznam parametrov ter izhodne vrednosti.

Da dosežemo, da PrepareLunch pogodba lahko kliče funkcijo makeSandwich, moramo slediti naslednjim korakom:
1. Definiramo interface FastFood pogodbe tako, da prilepimo spodnjo kodo v FastFoodInterface.sol:

``` Solidity
pragma solidity 0.5.0;

interface FastFoodInterface {
   function makeSandwich(string calldata _fillingA, string calldata _fillingB) external;
}
```

2. Nato moramo naložiti vsebino pogodbe ./FastFoodInterface.sol v PrepareLaunch pogodbo.

3. Na koncu moramo posodobiti še naslednje:

``` Solidity
fastFoodInstance = FastFoodInterface(_address);

```

Zdaj lahko pogodba PrepareLunch kliče makeSandwich funkcijo iz FastFood pogodbe.
Končna pogodba PrepareLunch naj bi izgledala takole:

``` Solidity
pragma solidity 0.5.0;
import "./FastFoodInterface.sol";

contract PrepareLunch {

  FastFoodInterface private fastFoodInstance;

  function instantiateFastFoodContract (address _address) public {
    fastFoodInstance = FastFoodInterface(_address);
    fastFoodInstance.makeSandwich("sliced ham", "pickled veggies");
  }
}
```

Uporabimo idejo zgornjega zgleda, da bomo lahko iz caller pametne pogodbe klicali updateEthPrice funkcijo 
iz oracle pametne pogodbe.
Spodnji kodi lahko prilepimo v naši datoteki CallerContract.sol in EthPriceOracleInterface.sol, ki
se nahajata v mapi caller.




``` Solidity
pragma solidity 0.5.0;
import "./EthPriceOracleInterface.sol";

contract CallerContract {
    EthPriceOracleInterface private oracleInstance;
    address private oracleAddress;
    function setOracleInstanceAddress (address _oracleInstanceAddress) public {
      oracleAddress = _oracleInstanceAddress;
      oracleInstance = EthPriceOracleInterface(oracleAddress);
    }
}

```


``` Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
interface EthPriceOracleInterface {
  function getLatestEthPrice() external returns (uint256);
}

```

# Modifier

Da naredimo našo pogodbo bolj varno, uporabimo modifier. Modifier je del kode, ki spremeni obnašanje funkcij. Na primer, lahko preverimo ali je določen pogoj izpolnjen pred izvajanjem posameznih funkcij. Sledili bom naslednjim korakom:

1. Uvozili bom vsebino iz pogodbe Ownable iz knjižice OpenZeppelin. 
2. Našo pogodbo bomo spremenili tako, da bo dedovala s pogodbe Ownable.
3. Spremenili bomo definicijo setOracleInstanceAddress funkcije tako, da bo uporabljala onlyOwner modifier.

Oglejmo si modifier še na preprostem primeru. V spodnjem primeru se onlyMe modifier zažene, preden se zažene koda v doSomething funkciji.


``` Solidity
contract MyContract {
  function doSomething public onlyMe {
    // do something
  }
}
```

Najlažje bo, da kodo v CallerContract.sol popravimo, da bo izgledala tako:

``` Solidity

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "./EthPriceOracleInterface.sol";
import "openzeppelin-solidity/contracts/access/Ownable.sol";

contract CallerContract is Ownable {
    EthPriceOracleInterface private oracleInstance;
    address private oracleAddress;
    event newOracleAddressEvent(address oracleAddress);
    function setOracleInstanceAddress (address _oracleInstanceAddress) public onlyOwner {
      oracleAddress = _oracleInstanceAddress;
      oracleInstance = EthPriceOracleInterface(oracleAddress);
      emit newOracleAddressEvent(oracleAddress);
    }
}

```
Opomba: če imamo problem z uvozom pogodbe Ownable, je rešitev lahko ta, da gremo v nastavitve našega urejevalnika in nastavimo Solidity: Package Default Dependencies Contracts Directory na contracts namesto prazno polje. Nato v poti pri uvozu odstranimo contracts.

# Uporaba Mappings za beleženje requestov

Zdaj si poglejmo, kako se ETH cena posodablja. Da zaženemo posodobitev ETH cene, bi morala pametna pogodba klicati getLatestEthPruce funkcijo iz oraklja. Ta funkcija ne more kar vrniti takega podatka. Namesto tega nam vrne edinstven id za vsak request. Potem orakelj potegne informacijo o ceni Eth iz Binance API, nato pa zažene callback funkcijo, ki posodobi Eth ceno v caller pogodbi. 

Vsak uporabnik naše dapp aplikacije lahko zažene operacijo, ki bo potrebovala caller pogodbo, da bo lahko naredila request, da posodobi ceno Eth. Ker klicatelj nima nobene kontrole nad tem, kdaj bo dobil odziv, moramo najti način, da bomo lahko beležili zgodovino vseh čakajočih requestov. S tem se bomo lahko prepričali, da bo vsak klic callback funkcije povezan s pravim requestom.

V naši pogodbi bomo uporabljali mapping imenovan myRequests. Mapping je v bistvu hash tabela, v kateri vsi ključi obstajajo. Sprva se vse vrednosti nastavi na privzete vrednosti tipa, ki smo ga podali.

Mapping lahko definiramo nekako takole:

``` Solidity
mapping(address => uint) public balances;
```

Kar smo zgoraj dejansko naredili je, da smo nastavili stanje vsem možnim računom na 0. Zakaj 0? Ker je to privzeta vrednost za uint.

Zgornji razmislek lahko uporabimo na naši pogodbi tako, da jo popravimo, da izgleda takole:

``` Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "./EthPriceOracleInterface.sol";
import "openzeppelin-solidity/access/Ownable.sol";

contract CallerContract is Ownable {
    EthPriceOracleInterface private oracleInstance;
    address private oracleAddress;
    mapping(uint256=>bool) myRequests;
    event newOracleAddressEvent(address oracleAddress);
    event ReceivedNewRequestIdEvent(uint256 id);
    function setOracleInstanceAddress (address _oracleInstanceAddress) public onlyOwner {
      oracleAddress = _oracleInstanceAddress;
      oracleInstance = EthPriceOracleInterface(oracleAddress);
      emit newOracleAddressEvent(oracleAddress);
    }
    function updateEthPrice() public {
      uint256 id = oracleInstance.getLatestEthPrice();
      myRequests[id] = true;
      emit ReceivedNewRequestIdEvent(id);
    }
}

```

# Callback funkcija

Naša caller pogodba je že skoraj končana, moramo pa še nekaj popraviti. Klicanje Binance public API je asinhrona operacija, zato mora caller pogodba podati callback funkcijo, ki jo bo orakelj lahko klical pozneje, ko bo prenesel Eth ceno. 

Tako naj bi delovala callback funkcija:
- Najprej bi se hoteli prepričati, da se funkcijo lahko kliče samo za veljaven id. Za ta namen bomo uporabili require ukaz. 
- Ko vemo, da je id veljaven, ga lahko odstranimo iz myRequests mappinga. (To bomo naredili tako, da bomo uporabili nekaj podobnega kot delete MojMapping[ključ].
- Na koncu mora naša funkcija še pognati event, da obvesti front-end, da je bila cena posodobljena.

Zgornji razmislek lahko implementiramo takole:

``` Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "./EthPriceOracleInterface.sol";
import "openzeppelin-solidity/access/Ownable.sol";

contract CallerContract is Ownable {
    uint256 private ethPrice;
    EthPriceOracleInterface private oracleInstance;
    address private oracleAddress;
    mapping(uint256=>bool) myRequests;
    event newOracleAddressEvent(address oracleAddress);
    event ReceivedNewRequestIdEvent(uint256 id);
    event PriceUpdatedEvent(uint256 ethPrice, uint256 id);
    function setOracleInstanceAddress (address _oracleInstanceAddress) public onlyOwner {
      oracleAddress = _oracleInstanceAddress;
      oracleInstance = EthPriceOracleInterface(oracleAddress);
      emit newOracleAddressEvent(oracleAddress);
    }
    function updateEthPrice() public {
      uint256 id = oracleInstance.getLatestEthPrice();
      myRequests[id] = true;
      emit ReceivedNewRequestIdEvent(id);
    }
    function callback(uint256 _ethPrice, uint256 _id) public {
      require(myRequests[_id], "This request is not in my pending list.");
      ethPrice = _ethPrice;
      delete myRequests[_id];
      emit PriceUpdatedEvent(_ethPrice, _id);
    }

}
```

# onlyOracle Modifier

Preden zaključimo s callback funkcijo, se moramo prepričati, da jo lahko kliče samo oracle pogodba. Zdaj bomo naredili modifier, ki bo preprečil drugim pogodbam, da bi lahko klicale callback funkcijo.

Spomnimo se, da smo že shranili naslov oraklja v spremenljivko, imenovano oracleAddress. Zato mora modifier samo preveriti, ali je naslov, ki kliče to pogodbo, res naslov oracleAddress. Za ta namen bomo uporabili msg.sender.

Našo pogodbo lahko popravimo, da izgleda tako:

``` Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "./EthPriceOracleInterface.sol";
import "openzeppelin-solidity/access/Ownable.sol";

contract CallerContract is Ownable {
    uint256 private ethPrice;
    EthPriceOracleInterface private oracleInstance;
    address private oracleAddress;
    mapping(uint256=>bool) myRequests;
    event newOracleAddressEvent(address oracleAddress);
    event ReceivedNewRequestIdEvent(uint256 id);
    event PriceUpdatedEvent(uint256 ethPrice, uint256 id);
    function setOracleInstanceAddress (address _oracleInstanceAddress) public onlyOwner {
      oracleAddress = _oracleInstanceAddress;
      oracleInstance = EthPriceOracleInterface(oracleAddress);
      emit newOracleAddressEvent(oracleAddress);
    }
    function updateEthPrice() public {
      uint256 id = oracleInstance.getLatestEthPrice();
      myRequests[id] = true;
      emit ReceivedNewRequestIdEvent(id);
    }
    function callback(uint256 _ethPrice, uint256 _id) public onlyOracle {
      require(myRequests[_id], "This request is not in my pending list.");
      ethPrice = _ethPrice;
      delete myRequests[_id];
      emit PriceUpdatedEvent(_ethPrice, _id);
    }
    modifier onlyOracle() {
      require(msg.sender == oracleAddress, "You are not authorized to call this function.");
      _;
    }
}

```

# getLatestEthPrice funkcija

Zdaj smo končno končali s pisanjem caller pogodbe in se lahko lotimo še orakelj pogodbe. Najprej si poglejmo, kaj naj bi ta pogodba počela.

Orakelj pogodba naj bi se obnašala kot nekakšen most, tako da bi omogočala caller pogodbam dostop do Eth cene. Da to dosežemo, bomo implementirali dve funkciji: getLatestEthPrice in seLatestEthPrice.

## getLatestEthPrice

Da omogočimo klicateljem, da sledijo svojemu requestu, mora getLatestEthPrice funkcija najprej izračunati id requesta in iz varnostnih razlogov bi moralo biti to število težko uganljivo. 

Eden od načinov, da v Solidityju dobimo "dovolj dobro" naključno število je, da uporabimo keccak256 funkcijo:
``` Solidity
uint randNonce = 0;
uint modulus = 1000;
uint randomNumber = uint(keccak256(abi.encodePacked(now, msg.sender, randNonce))) % modulus;

```

Zgornje vzame časovni žig (timestamp) od tega trenutka, msg.sender in nonce. Nato uporabi keccak256, da te podatke spremeni v naključen hash. Potem pa ta hash spremeni v uint. Na koncu še uporabi % modulus, da vzame samo zadnje tri števke. To nam da dovolj dobro naključno število med 0 in ostankom (modulus).

Zgornji razmislek lahko implementiramo tako, da v oracle mapi naredimo pogodbo EthPriceOracle.sol in vanjo prilepimo:
``` Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "openzeppelin-solidity/access/Ownable.sol";
import "./CallerContractInterface.sol";

contract EthPriceOracle is Ownable {
  uint private randNonce = 0;
  uint private modulus = 1000;
  mapping(uint256=>bool) pendingRequests;
  event GetLatestEthPriceEvent(address callerAddress, uint id);
  event SetLatestEthPriceEvent(uint256 ethPrice, address callerAddress);
  function getLatestEthPrice() public returns (uint256) {
    randNonce++;
    uint id = uint(keccak256(abi.encodePacked(block.timestamp, msg.sender, randNonce))) % modulus;
  }
}

```


Nato pa, kot pri caller pogodbi, naredimo še CallerContractInterface.sol v isti mapi in vanj prilepimo spodnje:
``` Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
interface CallerContractInterface {
  function callback(uint256 _ethPrice, uint256 id) external;
}

```

Zdaj pa bi radi implementirali še preprost sistem, ki beleži čakajoče requeste. Kot smo to naredili pri caller pogodbi, bomo tudi tu uporabili mapping. V tem primeru ga poimenujmo pendingRequests. 

getLatestEthPrice funkcija bi prav tako morala sprožiti dogodek in na koncu še vrniti id requesta. EthOracle pogodbo popravimo, da izgleda tako:

 ``` Solidity
 // SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "openzeppelin-solidity/access/Ownable.sol";
import "./CallerContractInterface.sol";

contract EthPriceOracle is Ownable {
  uint private randNonce = 0;
  uint private modulus = 1000;
  mapping(uint256=>bool) pendingRequests;
  event GetLatestEthPriceEvent(address callerAddress, uint id);
  event SetLatestEthPriceEvent(uint256 ethPrice, address callerAddress);
  function getLatestEthPrice() public returns (uint256) {
    randNonce++;
    uint id = uint(keccak256(abi.encodePacked(block.timestamp, msg.sender, randNonce))) % modulus;
    pendingRequests[id] = true;
    emit GetLatestEthPriceEvent(msg.sender, id);
    return id;
  }
}

 ``` 

## setLatestEthPrice funkcija

Smo že čisto pri koncu našega oraklja. Zdaj se bomo posvetili še setLatestEthPrice funkciji.

JavaScript komponente našega oraklja, ki jih bomo vključili v kodo pri naslednjem projektu Orakelj2, bodo pridobile Eth ceno iz Binance Public API in nato klicale setLatestEthPrice s tem, da ji bodo podale naslednje argumente:

- Eth ceno
- naslov pogodbe, ki je zagnala request
- id requesta

Najprej se mora naša funkcija prepričati, da jo lahko kliče samo lastnik. Potem, podobno kot prej, se mora naša funkcija prepričati, ali je id requesta veljaven, in v primeru, da je, ga odstraniti iz pendingRequests.

Naši orakelj pogodbi lahko dodamo spodnjo funkcijo, ki jo bomo dodelali kasneje.

``` Solidity 
  function setLatestEthPrice(uint256 _ethPrice, address _callerAddress, uint256 _id) public onlyOwner {
    require(pendingRequests[_id], "This request is not in my pending list.");
    delete pendingRequests[_id];
  }
```

# Oracle pogodba

Naša setLatestEthPrice funkcija je že skoraj končana. Kar bomo morali še narediti, je:

- Instantirati CallerContractsInstance.
-  Ko je caller pogodba instantirana, lahko izvedemo callback funkcijo ter ji podamo posodobljeno Eth ceno in id requesta.
-  Na koncu pa še sprožimo event, da obvesti front-end, da je bila cena uspešno posodobljena.

Našo oracle pogodbo moramo še popraviti, da bo izgledala kot spodaj, in smo končali.

``` Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "openzeppelin-solidity/access/Ownable.sol";
import "./CallerContractInterface.sol";


contract EthPriceOracle is Ownable {
  uint private randNonce = 0;
  uint private modulus = 1000;
  mapping(uint256=>bool) pendingRequests;
  event GetLatestEthPriceEvent(address callerAddress, uint id);
  event SetLatestEthPriceEvent(uint256 ethPrice, address callerAddress);
  function getLatestEthPrice() public returns (uint256) {
    randNonce++;
    uint id = uint(keccak256(abi.encodePacked(block.timestamp, msg.sender, randNonce))) % modulus;
    pendingRequests[id] = true;
    emit GetLatestEthPriceEvent(msg.sender, id);
    return id;
  }
  function setLatestEthPrice(uint256 _ethPrice, address _callerAddress, uint256 _id) public onlyOwner {
    require(pendingRequests[_id], "This request is not in my pending list.");
    delete pendingRequests[_id];
    CallerContractInterface = callerContractInstance;
    callerContractInstance = CallerContractInterface(_callerAddress);
    callerContractInstance.callback(_ethPrice, _id);
    emit SetLatestEthPriceEvent(_ethPrice, _callerAddress);
  }
}

```
