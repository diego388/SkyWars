
<?php
echo "PocketMine-MP plugin SkyWarsTeam v9.9.9
This file has been generated using DevTools v1.14.0 at Fri, 28 Aug 2020 17:51:31 +0000
----------------
";

if(extension_loaded("phar")){
	$phar = new \Phar(__FILE__);
	foreach($phar->getMetadata() as $key => $value){
		echo ucfirst($key) . ": " . (is_array($value) ? implode(", ", $value) : $value) . "\n";
	}
}

__HALT_COMPILER(); ?>
  
           �   a:9:{s:4:"name";s:11:"SkyWarsTeam";s:7:"version";s:5:"9.9.9";s:4:"main";s:15:"SkyWars\SkyWars";s:3:"api";s:5:"3.5.9";s:6:"depend";s:0:"";s:11:"description";s:0:"";s:7:"authors";s:0:"";s:7:"website";s:0:"";s:12:"creationDate";i:1598637091;}	   README.md�   #DI_�   ���j�      
   plugin.yml�   #DI_�   �_ ��         resources/skywars_settings.yml   #DI_   M��A�         src/SkyWars/Arena.php�K  #DI_�K  ��*�         src/SkyWars/ArenaScheduler.php�  #DI_�  n�h4�         src/SkyWars/EventListener.php�  #DI_�  P9n�         src/SkyWars/SkyWars.php�G  #DI_�G  ����         src/SkyWars/Team.php   #DI_   ���         src/SkyWars/npc/NPCEntity.phpZ  #DI_Z  �u�      %   src/SkyWars/scoreboard/Scoreboard.php�  #DI_�  0_�B�      # SkyWars
Minigame for PocketMine

  **Commands:**
* skywars createnpc: Creates join npc
* skywars create: [name] [slots] [type] Creates arena
* skywars setspawn: [spawn] sets player spawn
* type: solo/team
     

name: SkyWarsTeam
author: boi
api: 3.5.9
version: 9.9.9
main: SkyWars\SkyWars

commands:
   skywars:
     description: SkyWars Command
---
npc-position: 0:0:0:0:0<?php

namespace SkyWars;


use pocketmine\entity\effect\Effect;
use pocketmine\entity\effect\EffectInstance;
use pocketmine\item\Armor;
use pocketmine\item\enchantment\Enchantment;
use pocketmine\item\enchantment\EnchantmentInstance;
use pocketmine\item\Item;
use pocketmine\item\Sword;
use pocketmine\level\Level;
use pocketmine\level\Position;
use pocketmine\Player;
use pocketmine\tile\Chest;
use pocketmine\utils\Config;
use pocketmine\utils\TextFormat;
use pocketmine\math\Vector3;
use SkyWars\scoreboard\Scoreboard;

class Arena
{

    /** @var SkyWars $plugin */
    private $plugin;

    /** @var string $name */
    private $name;

    /** @var int $slots */
    public $slots;

    /** @var string $worldName */
    private $worldName;

    /** @var string $type */
    private $type;

    /** @var int $void */
    public $void;

    /** @var array $spawns */
    private $spawns = [];

    /** @var array $players */
    public $players = [];

    /** @var array $spectators */
    public $spectators = [];

    /** @var array $playerSpawns */
    private $playerSpawns = [];

    const STATE_COUNTDOWN = 0;
    const STATE_RUNNING = 1;

    /** @var int $GAME_STATE */
    public $GAME_STATE = self::STATE_COUNTDOWN;

    /** @var array $teams */
    private $teams = [];

    /** @var int $playersPerTeam */
    private $playersPerTeam = 2;

    /** @var int $requiredPlayers */
    private $requiredPlayers = 2;

    /** @var bool $starting */
    private $starting = false;

    /** @var int $startTime */
    public $startTime = 60;

    const TEAM_LIST = [
        'RED' => "§c",
        "GREEN" => "§a",
        "AQUA" => "§b",
        "YELLOW" => "§e",
        "BLUE" => "§9",
        "PINK" => "§b"
    ];

    /** @var int $time */
    private $time;

    /** @var int $lastRefill */
    private $lastRefill;


    /**
     * Arena constructor.
     * @param SkyWars $plugin
     * @param string $name
     * @param int $slots
     * @param string $worldName
     * @param string $type
     * @param int $void
     */
    public function __construct(SkyWars $plugin, string $name, int $slots = 0, string $worldName, string $type, int $void){
        $this->plugin = $plugin;
        $this->name = $name;
        $this->slots = $slots;
        $this->worldName = $worldName;
        $this->type = $type;
        $this->void = $void;


        if (!$this->reload($error)) {
            $logger = $this->plugin->getLogger();
            $logger->error("An error occured while reloading the arena: " . TextFormat::YELLOW . $this->SWname);
            $logger->error($error);
            $this->plugin->getServer()->getPluginManager()->disablePlugin($this->plugin);
        }

        if($this->type == "team"){
            $this->prepareTeams();
            $this->requiredPlayers = 4;

        }

    }

    /**
     * TEAM MODE ONLY
     */
    public function prepareTeams() : void{
        $teamCount = $this->slots / 2;
        $teams = [
            'RED' => "§c",
            "GREEN" => "§a",
            "AQUA" => "§b",
            "YELLOW" => "§e",
            "BLUE" => "§9",
            "PINK" => "§b"
        ];

        foreach(range(0, $teamCount - 1) as $int){
            echo $int . PHP_EOL;
            $teamColor = array_values($teams)[$int];
            $teamName = array_keys($teams)[$int];
            $this->teams[$int] = new Team($teamName, $teamColor);
        }

    }

    /**
     * @param Player $player
     */
    public function selectTeam(Player $player) : void{
        $top = $this->plugin->max_attribute_in_array($this->teams);
        foreach($this->teams as $team){
            if(count($team->getPlayers()) == $top && count($team->getPlayers()) !== $this->playersPerTeam){
                $team->addPlayer($player);
                return;
            }else{
                if(count($team->getPlayers()) !== $this->playersPerTeam){
                    $team->addPlayer($player);
                    return;
                }
            }
        }
    }

    /**
     * @return string
     */
    public function getType() : string{
        return $this->type;
    }

    /**
     * @return int
     */
    public function getLastRefill() : int{
        return $this->lastRefill;
    }

    /**
     * @return string
     */
    public function getName() : string{
        return $this->name;
    }

    /**
     * @param Player $player
     * @param int $spawn
     * @throws \InvalidStateException
     */
    public function setSpawn(Player $player, int $spawn) : void{
        if($spawn > $this->slots){
            $player->sendMessage(TextFormat::RED . "This arena got only " . $this->slots . " spawns available");
            return;
        }

        $config = new Config($this->plugin->getDataFolder() . "arenas/" . $this->name . "/settings.yml", Config::YAML);
        if (empty($config->get("spawns", []))) {
            $config->set("spawns", array_fill(1, $this->type == "team" ? $this->slots / 2 : $this->slots, [
                "x" => "n.a",
                "y" => "n.a",
                "z" => "n.a",
                "yaw" => "n.a",
                "pitch" => "n.a"
            ]));
        }
        $s = $config->get("spawns");
        $s[$spawn] = [
            "x" => floor($player->x),
            "y" => floor($player->y),
            "z" => floor($player->z),
            "yaw" => $player->yaw,
            "pitch" => $player->pitch
        ];

        $config->set("spawns", $s);
        $config->save();
        $this->spawns = $s;

        $player->sendMessage(TextFormat::GREEN . "Spawn " . TextFormat::YELLOW . $spawn . TextFormat::GREEN . " has been set");
    }

    /**
     * @param null $error
     * @return bool
     */
    private function reload(&$error = null) : bool
    {
        //Map reset
        if (!is_file($file = $this->plugin->getDataFolder() . "arenas/" . $this->name . "/" . $this->worldName . ".tar") && !is_file($file = $this->plugin->getDataFolder() . "arenas/" . $this->name . "/" . $this->worldName . ".tar.gz")) {
            $error = "Cannot find world backup file $file";
            return false;
        }

        $server = $this->plugin->getServer();

        if ($server->isLevelLoaded($this->worldName)) {
            $server->unloadLevel($server->getLevelByName($this->worldName));
        }

        $tar = new \PharData($file);
        $tar->extractTo($server->getDataPath() . "worlds/" . $this->worldName, null, true);

        $server->loadLevel($this->worldName);
        $server->getLevelByName($this->worldName)->setAutoSave(false);

        $config = new Config($this->plugin->getDataFolder() . "arenas/" . $this->name . "/settings.yml", Config::YAML, [//TODO: put descriptions
            "name" => $this->name,
            "slot" => $this->slots,
            "world" => $this->worldName,
            "gameType" => $this->type,
            "void_Y" => $this->void,
            "spawns" => []
        ]);

        $this->name = $config->get("name");
        $this->slots = (int) $config->get("slot");
        $this->worldName = $config->get("world");
        $this->type = $config->get('gameType');
        $this->spawns = $config->get("spawns");
        $this->void = (int) $config->get("void_Y");

        $this->players = [];
        $this->GAME_STATE = self::STATE_COUNTDOWN;

        $this->time = 0;
        $this->lastRefill = 0;
        return true;
    }

    /**
     * @return string
     */
    public function getWorldName() : string{
        return $this->worldName;
    }

    /**
     * @param Player $player
     */
    public function join(Player $player) : void{
        if($this->GAME_STATE == self::STATE_RUNNING){
            $player->sendMessage(TextFormat::RED . "Sorry, this game is already running!");
            return;
        }

        if(count($this->players) >= $this->slots || empty($this->slots)){
            $player->sendMessage(TextFormat::RED . "Sorry, this game is full!");
            return;
        }

        $player->setHealth($player->getMaxHealth());
        $player->setFood($player->getMaxFood());
        $player->removeAllEffects();
        $player->getInventory()->clearAll();
        $player->getArmorInventory()->clearAll();

        $server = $this->plugin->getServer();
        $level = $server->getLevelByName($this->worldName);

        $tmp = null;
        if($this->type == "solo") {
            $tmp = array_shift($this->spawns);
            $player->teleport(new Position($tmp["x"] + 0.5, $tmp["y"], $tmp["z"] + 0.5, $level), $tmp["yaw"], $tmp["pitch"]);
        }else{
            $this->selectTeam($player);
            $playerTeam = null;
            foreach($this->teams as $team){
                if(in_array($player->getName(), array_keys($team->players))){
                    $playerTeam = $team;
                }
            }
            $teamSpawn = array_search($playerTeam->getName(), array_keys(self::TEAM_LIST));
            $tmp = $this->spawns[$teamSpawn + 1];
            $player->teleport(new Position($tmp["x"] + 0.5, $tmp["y"], $tmp["z"] + 0.5, $level), $tmp["yaw"], $tmp["pitch"]);
            $player->sendMessage(TextFormat::GRAY . "You've joined team " . $playerTeam->getColor() . $playerTeam->getName());

        }
        $this->playerSpawns[$player->getRawUniqueId()] = $tmp;
        $this->plugin->setPlayerArena($player, $this->getName());
        $this->players[$player->getName()] = $player;
        $player->setImmobile(true);

        $this->broadcastMessage(TextFormat::GREEN . $player->getName() . " " . TextFormat::AQUA . "has joined the game " . TextFormat::YELLOW . "(" . count($this->players) . "/" . $this->slots . ")");

        Scoreboard::sendBoard($player, $this->plugin->getBoardFormat($this), "skywars");

    }

    /**
     * @param Player $player
     * @return bool
     */
    public function inArena(Player $player) : bool{
        return isset($this->players[$player->getName()]);
    }

    /**
     * @param string $message
     */
    public function broadcastMessage(string $message) : void{
        foreach($this->players as $player){
            $player->sendMessage($message);
        }
    }

    public function start() : void{
        $this->starting = false;
        $this->broadcastMessage(TextFormat::GREEN . "Game has started!");

        foreach($this->players as $player){
            $player->setImmobile(false);
        }
        $this->GAME_STATE = self::STATE_RUNNING;
        $this->time = 0;

        $this->refillChests();
    }

    /**
     * @return array
     */
    public function getAlivePlayers() : array{
        $alivePlayers = [];
        foreach($this->players as $player){
            if(isset($this->spectators[$player->getName()]))continue;
            $alivePlayers[$player->getName()] = $player;
        }
        return $alivePlayers;
    }

    /**
     * @param Player $player
     */
    public function killPlayer(Player $player) : void{
        if((count($this->getAlivePlayers()) - 1) > 1){
            $player->addTitle(TextFormat::BOLD . TextFormat::AQUA . "You lost!", TextFormat::YELLOW . "You are now spectating");
            $player->setGamemode(Player::SURVIVAL);
            $player->teleport(new Vector3($player->x, $this->void + 15, $player->z));
            $player->setFlying(true);
            foreach($this->getAlivePlayers() as $p){
                if(!$player instanceof Player)return;
                $p->hidePlayer($player);
            }
        }else{
            $this->spectators[$player->getName()] = $player;
            $player->teleport($this->plugin->getServer()->getDefaultLevel()->getSafeSpawn());
            Scoreboard::removeBoard($player, "skywars");
        }


    }

    /**
     * @param bool $force
     */
    public function stop(bool $force = false) : void{
        foreach($this->players as $player){
            $is_winner = !$force && in_array($player->getName(), array_keys($this->getAlivePlayers()));
            $this->removePlayer($player);

            if($is_winner){
                $this->plugin->getServer()->broadcastMessage(TextFormat::GREEN . $player->getName() . " " . TextFormat::AQUA . "has won SurvivalGames game on arena " . TextFormat::GREEN . $this->getName());
            }
        }


        $this->reload();
    }


    /**
     * @param Player $player
     */
    public function removePlayer(Player $player, bool $left = false, bool $spectate = false) : void{
        if($this->quit($player, $left, $spectate)){
            $player->setGamemode($this->plugin->getServer()->getDefaultGamemode());
            if(!$spectate){
                $player->setHealth(20);
                $player->setFood(20);
                $player->teleport($this->plugin->getServer()->getDefaultLevel()->getSafeSpawn());
            }elseif($this->GAME_STATE !== self::STATE_COUNTDOWN && count($this->getAlivePlayers()) > 1){
                $player->setGamemode(Player::SPECTATOR);
                foreach($this->getAlivePlayers() as $p){
                    $p->hidePlayer($player);
                }
                $player->addTitle(TextFormat::AQUA . "You lost!", TextFormat::YELLOW . "Type /hub to quit");
            }
        }
    }

    /**
     * @param Player $player
     * @param bool $left
     * @param bool $spectate
     * @return bool
     */
    public function quit(Player $player, bool $left = false, bool $spectate = false) : bool{
        if($this->GAME_STATE == self::STATE_COUNTDOWN){
            $player->setImmobile(false);
            $this->spawns[] = $this->playerSpawns[$uuid = $player->getRawUniqueId()];
            unset($this->playerSpawns[$uuid]);
        }

        if(isset($this->spectators[$player->getName()]) && $this->GAME_STATE == self::STATE_RUNNING){
            foreach($this->players as $pl){
                $pl->showPlayer($player);
            }
        }

        $this->plugin->setPlayerArena($player, null);

        if($left){
            unset($this->players[$player->getName()]);
            $this->broadcastMessage(TextFormat::AQUA . $player->getName() . " " . TextFormat::YELLOW . "has left the game " . TextFormat::GREEN . "(" . count($this->getAlivePlayers()) . "/" . $this->slots . ")");
        }

        if($spectate && !isset($this->spectators[$player->getName()])){
            $this->spectators[$player->getName()] = $player;
            foreach($this->spectators as $spectator){
                $spectator->showPlayer($player);
            }
        }else{
            Scoreboard::removeBoard($player, "skywars");
        }
        return true;
    }

    public function refillChests() : void{
        $level = $this->plugin->getServer()->getLevelByName($this->getWorldName());
        if(!$level instanceof Level)return;
        $contents = $this->plugin->getChestContents();
        foreach($level->getTiles() as $tile){
            if($tile instanceof Chest){
                $inventory = $tile->getInventory();
                $inventory->clearAll();

                if (empty($contents)) {
                    $contents = $this->plugin->getChestContents();
                }

                foreach (array_shift($contents) as $key => $val) {
                    $item = Item::get($val[0], 0, $val[1]);
                    $item = $this->enchantItem($item);
                    $inventory->setItem($key,$item , false);
                }

                $inventory->sendContents($inventory->getViewers());
            }
        }
        $this->lastRefill = time();
    }

    /**
     * @param $item
     * @return Item
     */
    public function enchantItem($item) : Item{
        $armorEnchantments = [
            Enchantment::PROTECTION => 4,
            Enchantment::FIRE_PROTECTION => 4,
            Enchantment::THORNS => 4
        ];

        $swordEnchantments = [
            Enchantment::FIRE_ASPECT => 2,
            Enchantment::SHARPNESS => 5,
            Enchantment::KNOCKBACK => 2
        ];

        a:
        if($item instanceof Armor){
            $enchantment = array_rand($armorEnchantments);
            if($b = rand(1,2) == 2){
                $item->addEnchantment(new EnchantmentInstance(Enchantment::getEnchantment($enchantment), $armorEnchantments[$enchantment]));
            }
            if($b == 2 && rand(1,2) == 2){
                goto a;//second enchantment
            }
        }
        if($item instanceof Sword){
            $enchantment = array_rand($swordEnchantments);
            if($b = rand(1,2) == 2){
                $item->addEnchantment(new EnchantmentInstance(Enchantment::getEnchantment($enchantment), $swordEnchantments[$enchantment]));
            }
            if($b == 2 && rand(1,2) == 2){
                goto a;//second enchantment
            }
        }
        return $item;
    }

    private $waitingDotState = 1;

    public function tick() : void{
        switch($this->GAME_STATE){
            case self::STATE_COUNTDOWN;
            if(count($this->players) < $this->requiredPlayers){
                foreach($this->players as $player){
                    $player->sendTip(TextFormat::YELLOW . "Waiting for players (" . TextFormat::AQUA . ($this->requiredPlayers - count($this->players)) . TextFormat::YELLOW . ")");
                    $this->waitingDotState++;
                    if($this->waitingDotState > 3)$this->waitingDotState = 1;
                }
            }else{
                if(!$this->starting){
                    $this->starting = true;
                    $this->broadcastMessage(TextFormat::GREEN . "Countdown started!");
                }
            }

            if($this->starting){
                $this->startTime--;
                foreach($this->players as $player){
                    $player->sendTip(TextFormat::AQUA . "Starting in " . TextFormat::YELLOW . gmdate("i:s", $this->startTime));
                }
                if(count($this->players) >= $this->slots - 2 && $this->startTime < 0){
                    $this->startTime = 0;
                }
                if($this->startTime == 0){
                     $this->start();
                }
            }
            break;
            case self::STATE_RUNNING;
            $playerCount = count($this->getAlivePlayers());
            if($playerCount < 2){
                $this->stop();
            }

            if($this->time % 240 == 0){
               $this->refillChests();
               foreach($this->players as $player){
                   $player->addTitle(TextFormat::GREEN . "Chests has been refilled!");
               }
            }
            break;
        }
        foreach($this->players as $player){
            Scoreboard::renameBoard("skywars", $this->plugin->getBoardFormat($this), $player);
        }
        $this->time++;
    }

}
<?php

namespace SkyWars;


use pocketmine\entity\Human;
use pocketmine\scheduler\Task;
use pocketmine\utils\TextFormat;

class ArenaScheduler extends Task
{

    /** @var SkyWars $plugin */
    private $plugin;

    const NPC_PREFIX = TextFormat::BOLD.TextFormat::AQUA."TAP TO JOIN";

    /**
     * ArenaScheduler constructor.
     * @param SkyWars $plugin
     */
    public function __construct(SkyWars $plugin)
    {
        $this->plugin = $plugin;
    }

    /**
     * @param int $currentTick
     */
    public function onRun(int $currentTick)
    {
         foreach($this->plugin->arenas as $arena){
             $arena->tick();
         }

    }

}<?php

namespace SkyWars;

use pocketmine\event\block\BlockBreakEvent;
use pocketmine\event\block\BlockPlaceEvent;
use pocketmine\event\entity\EntityDamageByEntityEvent;
use pocketmine\event\entity\EntityDamageEvent;
use pocketmine\event\entity\EntityLevelChangeEvent;
use pocketmine\event\Listener;
use pocketmine\event\player\PlayerMoveEvent;
use pocketmine\event\player\PlayerQuitEvent;
use pocketmine\event\server\DataPacketReceiveEvent;
use pocketmine\network\mcpe\protocol\ModalFormRequestPacket;
use pocketmine\network\mcpe\protocol\ModalFormResponsePacket;
use pocketmine\Player;
use pocketmine\utils\TextFormat;
use SkyWars\npc\NPCEntity;

class EventListener implements Listener
{

    /** @var SkyWars $plugin */
    private $plugin;

    /**
     * EventListener constructor.
     * @param SkyWars $plugin
     */
    public function __construct(SkyWars $plugin)
    {
        $this->plugin = $plugin;
    }

    /**
     * @param EntityDamageEvent $event
     */
    public function onEntityDamage(EntityDamageEvent $event) : void{
        $entity = $event->getEntity();
        if($entity instanceof Player) {
            $arena = $this->plugin->getPlayerArena($entity);
            if ($arena == null) return;


            if ($arena->GAME_STATE == Arena::STATE_RUNNING) {
                if ($event->getFinalDamage() >= $entity->getHealth()) {
                    $event->setCancelled();
                    if (!isset($arena->spectators[$entity->getName()])) {
                        $arena->removePlayer($entity, false, true);
                        $this->plugin->sendDeathMessage($arena, $event->getEntity());
                    }
                }
            }
        }
    }

    /**
     * @var array
     */
    private $arenaOrder = [];

    /**
     * @param DataPacketReceiveEvent $event
     */
    public function onPacketReceive(DataPacketReceiveEvent $event) : void{
        $player = $event->getPlayer();
        $packet = $event->getPacket();

        if($packet instanceof ModalFormResponsePacket){
            if($packet->formId == 599){
                $data = json_decode($packet->formData);
                if(is_null($data)){
                    return;
                }
                switch($data){
                    case 0;
                        $arena = $this->plugin->findBestArena();
                        $arena->join($player);
                        break;
                    case 1;
                        $formData = [];
                        $formData['title'] = "Arena List";
                        $formData['content'] = "";
                        $formData['type'] = "form";
                        $a = 0;
                        foreach($this->plugin->arenas as $arena){
                            $formData['buttons'][] = ['text' => $arena->getName() . " " . TextFormat::YELLOW . count($arena->players) . "/" . $arena->slots . "\n" . TextFormat::GRAY . "Type: " . TextFormat::BOLD . TextFormat::GREEN . ucfirst($arena->getType())];
                            $this->arenaOrder[$player->getName()][$a] = $arena->getName();
                            $a++;
                        }
                        $packet = new ModalFormRequestPacket();
                        $packet->formId = 600;
                        $packet->formData = json_encode($formData);
                        $player->dataPacket($packet);
                        break;
                }
            }elseif($packet->formId == 600){
                $formData = json_decode($packet->formData);
                if(is_null($formData)){
                    return;
                }

                if(in_array(intval($formData), range(0, count($this->plugin->arenas)))){
                    $this->plugin->arenas[array_values($this->arenaOrder[$player->getName()])[intval($packet->formData)]]->join($player);
                }

            }
        }
    }

    /**
     * @param PlayerQuitEvent $event
     */
    public function onQuit(PlayerQuitEvent $event) : void{
        $player = $event->getPlayer();
        $arena = $this->plugin->getPlayerArena($player);
        if($arena !== null){
            $arena->removePlayer($player, true, false);
        }
    }

    /**
     * @param EntityLevelChangeEvent $event
     */
    public function onLevelChange(EntityLevelChangeEvent $event) : void{
        $player = $event->getEntity();
        if(!$player instanceof Player)return;
        $target = $event->getTarget();
        $arena = $this->plugin->getPlayerArena($player);
        if($arena !== null){
            if($arena->getWorldName() !== $target->getFolderName()){
                $arena->removePlayer($player, true, false);
            }
        }
    }

    /**
     * @param BlockBreakEvent $event
     */
    public function onBlockBreak(BlockBreakEvent $event) : void{
        $player = $event->getPlayer();
        $arena = $this->plugin->getPlayerArena($player);
        if($arena !== null && $arena->GAME_STATE == Arena::STATE_COUNTDOWN){
            $event->setCancelled();
        }
    }

    /**
     * @param BlockPlaceEvent $event
     */
    public function onBlockPlace(BlockPlaceEvent $event) : void{
        $player = $event->getPlayer();
        $arena = $this->plugin->getPlayerArena($player);
        if($arena !== null && $arena->GAME_STATE == Arena::STATE_COUNTDOWN){
            $event->setCancelled();
        }
    }

    /**
     * @param PlayerMoveEvent $event
     */
    public function onMove(PlayerMoveEvent $event) : void{
        $player = $event->getPlayer();
        $arena = $this->plugin->getPlayerArena($player);
        if($arena !== null && in_array($player->getName(), array_keys($arena->getAlivePlayers()))){
            if($player->getY() < $arena->void){
                $arena->removePlayer($player, false, true);
                $this->plugin->sendDeathMessage($arena, $player, EntityDamageEvent::CAUSE_VOID);
            }
        }
    }





}<?php

namespace SkyWars;


use pocketmine\block\Block;
use pocketmine\command\Command;
use pocketmine\command\CommandSender;
use pocketmine\entity\Entity;
use pocketmine\entity\Human;
use pocketmine\event\entity\EntityDamageEvent;
use pocketmine\item\Item;
use pocketmine\level\Level;
use pocketmine\Player;
use pocketmine\plugin\PluginBase;
use pocketmine\utils\TextFormat;
use pocketmine\math\Vector3;
use SkyWars\npc\NPCEntity;

class SkyWars extends PluginBase
{

    /** @var Arena[] $arenas */
    public $arenas = [];

    /** @var array $settings */
    private $settings;

    /** @var array $playerArenas */
    private $playerArenas = [];

    /** @var Entity $NPCEntity */
    private $NPCEntity;

    /** @var SkyWars $instance */
    public static $instance;

    public function onEnable() : void
    {

        foreach ($this->getResources() as $resource) {
            $this->saveResource($resource->getFilename());
        }

        $this->settings = yaml_parse_file($this->getDataFolder() . "skywars_settings.yml");
        $this->loadArenas();

        $this->getServer()->getPluginManager()->registerEvents(new EventListener($this), $this);
        $this->getScheduler()->scheduleRepeatingTask(new ArenaScheduler($this), 20);

        Entity::registerEntity(NPCEntity::class, true);

        self::$instance = $this;



    }

    /**
     * @param Player $player
     */
    public function spawnNPC(Player $player) : void{
        $nbt = Entity::createBaseNBT($player, null, $player->getYaw(), $player->getPitch());
        $skinTag = $player->namedtag->getCompoundTag("Skin");
        assert($skinTag !== null);
        $nbt->setTag(clone $skinTag);

        $nametag = TextFormat::BOLD . TextFormat::AQUA . "SkyWars" . TextFormat::RESET . "\n".
                   TextFormat::YELLOW . "0 Playing";

        $entity = Entity::createEntity("NPCEntity", $player->getLevel(), $nbt);
        $entity->setNameTag($nametag);
        $entity->spawnToAll();
    }

    public function loadArenas() : void{
        $base_path = $this->getDataFolder() . "arenas/";
        @mkdir($base_path);

        foreach (scandir($base_path) as $dir) {
            $dir = $base_path . $dir;
            $settings_path = $dir . "/settings.yml";

            if (!is_file($settings_path)) {
                continue;
            }

            $arena_info = yaml_parse_file($settings_path);

            $this->arenas[$arena_info["name"]] = new Arena(
                $this,
                (string) $arena_info["name"],
                (int) $arena_info["slot"],
                (string) $arena_info["world"],
                (string) $arena_info['gameType'],
                (int) $arena_info["void_Y"]
            );
        }
    }

    /**
     * @return array
     */
    public function getSettings() : array{
        return $this->settings;
    }

    /**
     * @param Player $player
     * @param null|string $arena
     */
    public function setPlayerArena(Player $player, ?string $arena) : void{
        if(is_null($arena)){
            unset($this->playerArenas[$player->getName()]);
            return;
        }

        $this->playerArenas[$player->getName()] = $arena;
    }

    /**
     * @param Player $player
     * @return null|Arena
     */
    public function getPlayerArena(Player $player) : ?Arena{
        return isset($this->playerArenas[$player->getName()]) ? $this->arenas[$this->playerArenas[$player->getName()]] : null;
    }

    /**
     * @param CommandSender $sender
     * @param Command $command
     * @param string $label
     * @param array $args
     * @return bool
     * @throws \InvalidStateException
     */
    public function onCommand(CommandSender $sender, Command $command, string $label, array $args): bool
    {
        if(strtolower($command->getName()) == "skywars"){
            if(empty($args[0]) || !$sender->isOp())return false;
            switch($args[0]){
                case "create";
                if(!$sender instanceof Player){
                    $sender->sendMessage(TextFormat::RED . "This command can be executed only in game!");
                    return false;
                }


                if(count($args) < 2){
                    $sender->sendMessage(TextFormat::AQUA . "Usage: " . TextFormat::WHITE . "/skywars create [name] [slots] [type]");
                    return false;
                }

                $arenaName = $args[1];
                $slots = intval($args[2]);
                $arenaType = $args[3];

                if($arenaType !== "solo" && $arenaType !== "team"){
                    $sender->sendMessage(TextFormat::RED . "Invalid arena type");
                    return false;
                }

                $level = $sender->getLevel();
                $levelName = $sender->getLevel()->getFolderName();

                if($this->getServer()->getDefaultLevel()->getFolderName() == $levelName){
                    $sender->sendMessage(TextFormat::RED . "You can't create arenas in default level!");
                    return false;
                }

                foreach($this->arenas as $arena => $arenaInstance){
                    if($arenaInstance->getWorldName() == $levelName){
                        $sender->sendMessage(TextFormat::RED . "There's already arena in this level!");
                        return false;
                    }
                }

                $sender->sendMessage(TextFormat::LIGHT_PURPLE . "Calculating minimum void in world '" . $levelName . "'...");

                $void_y = Level::Y_MAX;
                foreach ($level->getChunks() as $chunk) {
                    for ($x = 0; $x < 16; ++$x) {
                        for ($z = 0; $z < 16; ++$z) {
                            for ($y = 0; $y < $void_y; ++$y) {
                                $block = $chunk->getBlockId($x, $y, $z);
                                if ($block !== Block::AIR) {
                                    $void_y = $y;
                                    break;
                                }
                                }
                            }
                        }
                    }
                --$void_y;

                $sender->sendMessage(TextFormat::LIGHT_PURPLE . "Minimum void set to: " . $void_y);

                $sender->teleport($this->getServer()->getDefaultLevel()->getSpawnLocation());
                $this->getServer()->unloadLevel($level);
                unset($level);

                @mkdir($this->getDataFolder() . "arenas/" . $arenaName, 0755);
                $tar = new \PharData($this->getDataFolder() . "arenas/" . $arenaName . "/" . $levelName . ".tar");
                $tar->startBuffering();
                $tar->buildFromDirectory(realpath($sender->getServer()->getDataPath() . "worlds/" . $levelName));
                $tar->stopBuffering();

                $sender->sendMessage(TextFormat::LIGHT_PURPLE . "Backup of world '" . $levelName . "' created.");
                $this->getServer()->loadLevel($levelName);

                $this->arenas[$arenaName] = new Arena($this, $arenaName, $slots, $levelName, $arenaType, $void_y);
                $sender->sendMessage(TextFormat::GREEN . "Arena created");
                break;
                case "setspawn";
                if(!$sender instanceof Player){
                    $sender->sendMessage(TextFormat::RED . "This command can be executed only in game!");
                    return false;
                }
                if(empty($args[1])){
                    $sender->sendMessage(TextFormat::AQUA . "Usage: " . TextFormat::WHITE . "/skywars setspawn [spawn]");
                }
                $levelName = $sender->getLevel()->getFolderName();

                $arena = null;
                foreach($this->arenas as $name => $arenaInstance){
                    if($arenaInstance->getWorldName() == $levelName){
                        $arena = $arenaInstance;
                    }
                }

                if(is_null($arena)){
                    $sender->sendMessage(TextFormat::RED . "Arena not found in this level!");
                    return false;
                }

                $arena->setSpawn($sender, intval($args[1]));
                break;
                case "createnpc";
                $this->spawnNPC($sender);
                break;
            }
        }
        return false;
    }

    /**
     * @return null|Entity
     */
    public function getNPCEntity() : ?Entity{
        $level = $this->getServer()->getDefaultLevel();
        if(is_null($this->NPCEntity)) {
            foreach ($level->getEntities() as $entity) {
                if ($entity instanceof Player) return null;
                $name = explode("\n", $entity->getNameTag());
                if ($name[0] == TextFormat::GREEN . "SkyWars") {
                    $this->NPCEntity = $entity;
                    return $entity;
                }
            }
        }else{
            return $this->NPCEntity;
        }
        return null;
    }

    /**
     * @return Arena
     */
    public function findBestArena() : ?Arena{
        $newArenas = [];
        foreach($this->arenas as $arena){
            if($arena->GAME_STATE !== Arena::STATE_COUNTDOWN)continue;
            $newArenas[] = $arena;

        }
        $max = $this->max_attribute_in_array($newArenas);
        foreach($newArenas as $arena){
            if(count($arena->players) == $max){
                return $arena;
            }
        }
        return null;
    }

    /**
     * @param $data_points
     * @param string $value
     * @return int
     */
    public function max_attribute_in_array($data_points, $value='players') : int{
        $max=0;
        foreach($data_points as $point){
            if($max < (float)count($point->{$value})){
                $max = count($point->{$value});
            }
        }
        return $max;
    }

    public function getChestContents() : array//TODO: **rewrite** this and let the owner decide the contents of the chest
    {
        $items = [
            //ARMOR
            "armor" => [
                [
                    Item::LEATHER_CAP,
                    Item::LEATHER_TUNIC,
                    Item::LEATHER_PANTS,
                    Item::LEATHER_BOOTS
                ],
                [
                    Item::GOLD_HELMET,
                    Item::GOLD_CHESTPLATE,
                    Item::GOLD_LEGGINGS,
                    Item::GOLD_BOOTS
                ],
                [
                    Item::CHAIN_HELMET,
                    Item::CHAIN_CHESTPLATE,
                    Item::CHAIN_LEGGINGS,
                    Item::CHAIN_BOOTS
                ],
                [
                    Item::IRON_HELMET,
                    Item::IRON_CHESTPLATE,
                    Item::IRON_LEGGINGS,
                    Item::IRON_BOOTS
                ],
                [
                    Item::DIAMOND_HELMET,
                    Item::DIAMOND_CHESTPLATE,
                    Item::DIAMOND_LEGGINGS,
                    Item::DIAMOND_BOOTS
                ]
            ],

            //WEAPONS
            "weapon" => [
                [
                    Item::WOODEN_SWORD,
                    Item::WOODEN_AXE,
                ],
                [
                    Item::GOLD_SWORD,
                    Item::GOLD_AXE
                ],
                [
                    Item::STONE_SWORD,
                    Item::STONE_AXE
                ],
                [
                    Item::IRON_SWORD,
                    Item::IRON_AXE
                ],
                [
                    Item::DIAMOND_SWORD,
                    Item::DIAMOND_AXE
                ]
            ],

            //FOOD
            "food" => [
                [
                    Item::RAW_PORKCHOP,
                    Item::RAW_CHICKEN,
                    Item::MELON_SLICE,
                    Item::COOKIE
                ],
                [
                    Item::RAW_BEEF,
                    Item::CARROT
                ],
                [
                    Item::APPLE,
                    Item::GOLDEN_APPLE
                ],
                [
                    Item::BEETROOT_SOUP,
                    Item::BREAD,
                    Item::BAKED_POTATO
                ],
                [
                    Item::MUSHROOM_STEW,
                    Item::COOKED_CHICKEN
                ],
                [
                    Item::COOKED_PORKCHOP,
                    Item::STEAK,
                    Item::PUMPKIN_PIE
                ],
            ],

            //THROWABLE
            "throwable" => [
                [
                    Item::BOW,
                    Item::ARROW
                ],
                [
                    Item::SNOWBALL
                ],
                [
                    Item::EGG
                ]
            ],

            //BLOCKS
            "block" => [
                Item::STONE,
                Item::WOODEN_PLANKS,
                Item::COBBLESTONE,
                Item::DIRT
            ],

            //OTHER
            "other" => [
                [
                    Item::WOODEN_PICKAXE,
                    Item::GOLD_PICKAXE,
                    Item::STONE_PICKAXE,
                    Item::IRON_PICKAXE,
                    Item::DIAMOND_PICKAXE
                ],
                [
                    Item::STICK,
                    Item::STRING
                ]
            ]
        ];

        $templates = [];
        for ($i = 0; $i < 10; $i++) {//TODO: understand wtf is the stuff in here doing

            $armorq = mt_rand(0, 1);
            $armortype = $items["armor"][array_rand($items["armor"])];

            $armor1 = [$armortype[array_rand($armortype)], 1];
            if ($armorq) {
                $armortype = $items["armor"][array_rand($items["armor"])];
                $armor2 = [$armortype[array_rand($armortype)], 1];
            } else {
                $armor2 = [0, 1];
            }

            $weapontype = $items["weapon"][array_rand($items["weapon"])];
            $weapon = [$weapontype[array_rand($weapontype)], 1];

            $ftype = $items["food"][array_rand($items["food"])];
            $food = [$ftype[array_rand($ftype)], mt_rand(2, 5)];

            if (mt_rand(0, 1)) {
                $tr = $items["throwable"][array_rand($items["throwable"])];
                if (count($tr) === 2) {
                    $throwable1 = [$tr[1], mt_rand(10, 20)];
                    $throwable2 = [$tr[0], 1];
                } else {
                    $throwable1 = [0, 1];
                    $throwable2 = [$tr[0], mt_rand(5, 10)];
                }
                $other = [0, 1];
            } else {
                $throwable1 = [0, 1];
                $throwable2 = [0, 1];
                $ot = $items["other"][array_rand($items["other"])];
                $other = [$ot[array_rand($ot)], 1];
            }

            $block = [$items["block"][array_rand($items["block"])], 64];

            $contents = [
                $armor1,
                $armor2,
                $weapon,
                $food,
                $throwable1,
                $throwable2,
                $block,
                $other
            ];
            shuffle($contents);

            $fcontents = [
                mt_rand(0, 1) => array_shift($contents),
                mt_rand(2, 4) => array_shift($contents),
                mt_rand(5, 9) => array_shift($contents),
                mt_rand(10, 14) => array_shift($contents),
                mt_rand(15, 16) => array_shift($contents),
                mt_rand(17, 19) => array_shift($contents),
                mt_rand(20, 24) => array_shift($contents),
                mt_rand(25, 26) => array_shift($contents),
            ];

            $templates[] = $fcontents;
        }

        shuffle($templates);
        return $templates;
    }

    /**
     * @param Arena $arena
     * @param Player $player
     * @param int $forceCause
     */
    public function sendDeathMessage(Arena $arena, Player $player, int $forceCause = null) : void{
        $lastCause = $player->getLastDamageCause();
        switch($forceCause == null ? $lastCause->getCause() : $forceCause){
            case EntityDamageEvent::CAUSE_ENTITY_ATTACK;
            $killer = $lastCause->getDamager();
            $arena->broadcastMessage(TextFormat::GREEN . $player->getName() . " " . TextFormat::AQUA . "was killed by " . TextFormat::RED . $killer->getName() . " " . TextFormat::GREEN . "(" . count($arena->getAlivePlayers()) . "/" . $arena->slots . ")");
            break;
            case EntityDamageEvent::CAUSE_VOID;
            $arena->broadcastMessage(TextFormat::GREEN . $player->getName() . " " . TextFormat::AQUA . "was killed by  " . TextFormat::RED . "Void " . TextFormat::GREEN . "(" . count($arena->getAlivePlayers()) . "/" . $arena->slots . ")");
            break;
        }
    }

    /**
     * @param Arena $arena
     * @return string
     */
    public function getBoardFormat(Arena $arena) : string{
        $format =  [
            0 => TextFormat::BOLD . TextFormat::YELLOW . "SkyWars\n" . TextFormat::RESET.
                 TextFormat::AQUA . "Players: " . TextFormat::GREEN . count($arena->players) . "\n".
                 TextFormat::AQUA . "Start: " . TextFormat::GREEN . gmdate("i:s", $arena->startTime),
            1 => TextFormat::BOLD . TextFormat::YELLOW . "SkyWars\n" . TextFormat::RESET.
                 TextFormat::AQUA . "Players: " . TextFormat::GREEN . count($arena->getAlivePlayers()) . "\n".
                 TextFormat::AQUA . "Chest refill: " . TextFormat::GREEN . gmdate("i:s", round(($arena->getLastRefill() + 240) - time()))];
        return $format[$arena->GAME_STATE];
    }


}<?php

namespace SkyWars;


use pocketmine\Player;

class Team
{

    /** @var string $color */
    private $color;

    /** @var string $name */
    private $name;

    /** @var array $players */
    public $players = [];

    /**
     * Team constructor.
     * @param string $name
     * @param string $color
     */
    public function __construct(string $name, string $color)
    {
        $this->color = $color;
        $this->name = $name;
    }

    /**
     * @return array
     */
    public function getPlayers() : array{
        return $this->players;
    }

    /**
     * @param Player $player
     */
    public function addPlayer(Player $player) : void{
        $this->players[$player->getName()] = $player;
     }

    /**
     * @return string
     */
     public function getName() : string{
        return $this->name;
     }

    /**
     * @return string
     */
     public function getColor() : string{
         return $this->color;
     }

}<?php

namespace SkyWars\npc;


use pocketmine\entity\Human;
use pocketmine\event\entity\EntityDamageByEntityEvent;
use pocketmine\event\entity\EntityDamageEvent;
use pocketmine\network\mcpe\protocol\ModalFormRequestPacket;
use pocketmine\Player;
use pocketmine\utils\TextFormat;
use SkyWars\SkyWars;

class NPCEntity extends Human
{

    /**
     * @param int $tickDiff
     * @return bool
     */
    public function entityBaseTick(int $tickDiff = 1): bool
    {
        $playerCount = 0;
        foreach(SkyWars::$instance->arenas as $arena){
            $playerCount += count($arena->players);
        }

        $this->setNameTag(TextFormat::AQUA . TextFormat::BOLD . "SkyWars§r\n" . TextFormat::YELLOW . $playerCount . " Playing");
        return parent::entityBaseTick($tickDiff);
    }

    /**
     * @param EntityDamageEvent $source
     */
    public function attack(EntityDamageEvent $source): void
    {
        $source->setCancelled();
        if(!$source instanceof EntityDamageByEntityEvent)return;
        $damager = $source->getDamager();
        if(!$damager instanceof Player)return;
        $formData = [];
        $formData['title'] = "SkyWars Menu";
        $formData['type'] = "form";
        $arena = SkyWars::$instance->findBestArena();
        $formData['buttons'][] = ['text' => "Join current arena\n" . TextFormat::RESET . $arena->getName() . " " . TextFormat::YELLOW . count($arena->players) . "/" . $arena->slots . " " . TextFormat::GRAY . "Type: " . TextFormat::BOLD . TextFormat::GREEN . ucfirst($arena->getType())];
        $formData['buttons'][] = ['text' => 'Select arena'];
        $formData['content'] = "";
        $packet = new ModalFormRequestPacket();
        $packet->formId = 599;
        $packet->formData = json_encode($formData);
        $damager->dataPacket($packet);
    }


}
<?php

namespace SkyWars\scoreboard;


use pocketmine\entity\Entity;
use pocketmine\network\mcpe\protocol\RemoveObjectivePacket;
use pocketmine\network\mcpe\protocol\SetDisplayObjectivePacket;
use pocketmine\network\mcpe\protocol\SetScorePacket;
use pocketmine\network\mcpe\protocol\types\ScorePacketEntry;
use pocketmine\Player;

class Scoreboard
{

    /** @var int $line */
    private static $eid;
    /**
     * @param Player $player
     * @param string $objective
     */
    public static function removeBoard(Player $player, string $objective) {
        $pk = new RemoveObjectivePacket();
        $pk->objectiveName = $objective;
        $player->dataPacket($pk);
    }
    /**
     * @param Player $player
     * @param string $text
     * @param string|null $objectiveName
     */
    public static function sendBoard(Player $player, string $text, string $objectiveName = null) {
        if($objectiveName === null) $objectiveName = strtolower($player->getLevel()->getFolderName());
        $lines = explode(PHP_EOL, $text);
        $title = array_shift($lines);
        foreach (self::buildBoard($objectiveName, $title, implode(PHP_EOL, $lines)) as $packet) {
            $player->dataPacket($packet);
        }
    }

    /**
     * @param string $id
     * @param string $text
     * @param Player $player
     */
    public static function renameBoard(string $id, string $text, Player $player) : void{
        $pk1= new RemoveObjectivePacket();
        $pk1->objectiveName = $id;

        $packets[] = $pk1;

        $lines = explode(PHP_EOL, $text);
        foreach(self::buildBoard($id, array_shift($lines), implode(PHP_EOL, $lines)) as $pk){
            $packets[] = $pk;
        }
        foreach($packets as $packet){
            $player->dataPacket($packet);
        }
    }

    /**
     * @param string $id
     * @param string $title
     * @param string $text
     * @return array
     */
    public static function buildBoard(string $id, string $title, string $text) {
        $pk = new SetDisplayObjectivePacket();
        $pk->objectiveName = $id;
        $pk->displayName = $title;
        $pk->sortOrder = 0;
        $pk->criteriaName = "dummy";
        $pk->displaySlot = "sidebar";
        $packets[] = clone $pk;
        self::$eid = Entity::$entityCount++;
        $pk = new SetScorePacket();
        $pk->type = $pk::TYPE_CHANGE;
        $pk->entries = self::buildLines($id, $text);
        $packets[] = clone $pk;
        return $packets;
    }

    /**
     * @param string $id
     * @param string $text
     * @return ScorePacketEntry[] $lines
     */
    private static function buildLines(string $id, string $text): array {
        $texts = explode(PHP_EOL, $text);
        $lines = [];
        foreach ($texts as $line) {
            $entry = new ScorePacketEntry();
            $entry->score = count($lines);
            $entry->scoreboardId = count($lines);
            $entry->objectiveName = $id;
            $entry->entityUniqueId = self::$eid;
            $entry->type = ScorePacketEntry::TYPE_FAKE_PLAYER;
            $entry->customName = " " . $line . " "; // it seems better
            $lines[] = $entry;
        }
        return $lines;
    }

}xk��:�S9�PB�9bԏ   GBMB
