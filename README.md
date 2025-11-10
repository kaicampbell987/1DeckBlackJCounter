# 1DeckBlackJCounter
import subprocess, sys, importlib, os, time, json, re, asyncio
from datetime import timedelta
from typing import Any, Dict, List, Optional, Tuple
from uuid import uuid4
from collections import defaultdict

from uagents import Agent, Context, Protocol
from uagents_core.contrib.protocols.chat import ChatMessage, TextContent, StartSessionContent

# ============================================================================
# Card tracking system with JSON serialization
# ============================================================================

class CardZone:
    """Tracks cards in a specific zone (deck, player hand, dealer hand, discard)"""
    def __init__(self, name: str):
        self.name = name
        self.cards = defaultdict(int)
    
    def add_card(self, card: str, count: int = 1):
        """Add card(s) to this zone"""
        card = card.upper().strip()
        self.cards[card] += count
    
    def remove_card(self, card: str, count: int = 1) -> bool:
        """Remove card(s) from this zone. Returns False if not enough cards."""
        card = card.upper().strip()
        if self.cards[card] >= count:
            self.cards[card] -= count
            if self.cards[card] == 0:
                del self.cards[card]
            return True
        return False
    
    def get_all_cards(self) -> List[str]:
        """Get list of all cards in this zone"""
        result = []
        for card, count in self.cards.items():
            result.extend([card] * count)
        return result
    
    def clear(self) -> List[str]:
        """Remove all cards and return them"""
        cards = self.get_all_cards()
        self.cards.clear()
        return cards
    
    def total_cards(self) -> int:
        """Total number of cards in zone"""
        return sum(self.cards.values())
    
    def to_dict(self) -> dict:
        """Serialize to dictionary"""
        return {"name": self.name, "cards": dict(self.cards)}
    
    @staticmethod
    def from_dict(data: dict):
        """Deserialize from dictionary"""
        zone = CardZone(data["name"])
        zone.cards = defaultdict(int, data["cards"])
        return zone
    
    def __str__(self):
        if not self.cards:
            return f"{self.name}: Empty"
        card_list = []
        for card in sorted(self.cards.keys()):
            count = self.cards[card]
            card_list.append(f"{card}×{count}" if count > 1 else card)
        return f"{self.name} ({self.total_cards()}): {', '.join(card_list)}"

class GameState:
    """Manages all card zones and game state"""
    def __init__(self):
        self.deck = CardZone("Deck")
        self.dealer_hand = CardZone("Dealer Hand")
        self.discard = CardZone("Discard Pile")
        self.face_down = CardZone("Face Down")
        self.face_down_owners = {}
        self.players = {}
        self.next_player_number = 1
        self.initialize_deck()
    
    def initialize_deck(self):
        """Create a fresh 52-card deck"""
        self.deck.clear()
        for card in ['2', '3', '4', '5', '6', '7', '8', '9', '10', 'J', 'Q', 'K', 'A']:
            self.deck.add_card(card, 4)
    
    def to_dict(self) -> dict:
        """Serialize game state to dictionary"""
        return {
            "deck": self.deck.to_dict(),
            "dealer_hand": self.dealer_hand.to_dict(),
            "discard": self.discard.to_dict(),
            "face_down": self.face_down.to_dict(),
            "face_down_owners": self.face_down_owners,
            "players": {num: zone.to_dict() for num, zone in self.players.items()},
            "next_player_number": self.next_player_number
        }
    
    @staticmethod
    def from_dict(data: dict):
        """Deserialize game state from dictionary"""
        state = GameState()
        state.deck = CardZone.from_dict(data["deck"])
        state.dealer_hand = CardZone.from_dict(data["dealer_hand"])
        state.discard = CardZone.from_dict(data["discard"])
        state.face_down = CardZone.from_dict(data["face_down"])
        state.face_down_owners = data["face_down_owners"]
        state.players = {int(num): CardZone.from_dict(zone_data) for num, zone_data in data["players"].items()}
        state.next_player_number = data["next_player_number"]
        return state
    
    def add_player(self) -> str:
        """Add a new player and assign them a number"""
        player_num = self.next_player_number
        self.players[player_num] = CardZone(f"Player {player_num} Hand")
        self.next_player_number += 1
        return f"✅ Added Player {player_num}"
    
    def get_player_zone(self, identifier: str) -> tuple:
        """Get player zone from identifier like 'player', 'player1', 'player 1', '1'"""
        identifier = identifier.lower().strip()
        identifier = identifier.replace("player", "").strip()
        
        if not identifier:
            identifier = "1"
        
        try:
            player_num = int(identifier)
            if player_num in self.players:
                return (player_num, self.players[player_num])
            else:
                return (None, None)
        except ValueError:
            return (None, None)
    
    def deal_card(self, card: str, to_zone: str) -> str:
        """Move card from deck to specified zone (player/dealer)"""
        card = card.upper().strip()
        
        if not self.deck.remove_card(card):
            return f"❌ Card {card} not available in deck"
        
        to_zone_lower = to_zone.lower()
        
        if to_zone_lower == "dealer":
            self.dealer_hand.add_card(card)
            return f"✅ Dealt {card} to Dealer"
        else:
            player_num, player_zone = self.get_player_zone(to_zone)
            if player_zone:
                player_zone.add_card(card)
                return f"✅ Dealt {card} to Player {player_num}"
            else:
                self.deck.add_card(card)
                return f"❌ Invalid zone: {to_zone}. Use 'Player 1', 'Player 2', or 'Dealer'"
    
    def deal_face_down(self, to_zone: str) -> str:
        """Deal a face down card (moves from deck to face down zone)"""
        to_zone_lower = to_zone.lower()
        
        total_available = self.deck.total_cards() + self.face_down.total_cards()
        if total_available == 0:
            return "❌ No cards left in deck"
        
        owner_key = None
        if to_zone_lower == "dealer":
            owner_key = "dealer"
        else:
            player_num, _ = self.get_player_zone(to_zone)
            if player_num:
                owner_key = f"player{player_num}"
            else:
                return f"❌ Invalid zone: {to_zone}"
        
        self.face_down.add_card("FACEDOWN")
        
        if owner_key not in self.face_down_owners:
            self.face_down_owners[owner_key] = 0
        self.face_down_owners[owner_key] += 1
        
        display_name = "Dealer" if owner_key == "dealer" else f"Player {player_num}"
        return f"✅ Dealt 1 face down card to {display_name} (use 'Flip {display_name} <card>' to reveal)"
    
    def flip_card(self, who: str, card: str) -> str:
        """Flip a face down card and assign its value"""
        card = card.upper().strip()
        who_lower = who.lower()
        
        owner_key = None
        target_zone = None
        display_name = None
        
        if who_lower == "dealer":
            owner_key = "dealer"
            target_zone = self.dealer_hand
            display_name = "Dealer"
        else:
            player_num, player_zone = self.get_player_zone(who)
            if player_zone:
                owner_key = f"player{player_num}"
                target_zone = player_zone
                display_name = f"Player {player_num}"
            else:
                return f"❌ Invalid target: {who}"
        
        if owner_key not in self.face_down_owners or self.face_down_owners[owner_key] == 0:
            return f"❌ No face down card for {display_name}"
        
        if not self.deck.remove_card(card):
            return f"❌ Card {card} not available in deck"
        
        self.face_down.remove_card("FACEDOWN")
        self.face_down_owners[owner_key] -= 1
        if self.face_down_owners[owner_key] == 0:
            del self.face_down_owners[owner_key]
        
        target_zone.add_card(card)
        return f"✅ Flipped {display_name}'s face down card: {card}"
    
    def hit(self, card: str, who: str) -> str:
        """Add specified card to player or dealer hand"""
        card = card.upper().strip()
        
        if not self.deck.remove_card(card):
            return f"❌ Card {card} not available in deck (remaining: {self.deck.total_cards()})"
        
        who_lower = who.lower()
        
        if who_lower == "dealer":
            self.dealer_hand.add_card(card)
            return f"✅ Dealer hit: {card}"
        else:
            player_num, player_zone = self.get_player_zone(who)
            if player_zone:
                player_zone.add_card(card)
                return f"✅ Player {player_num} hit: {card}"
            else:
                self.deck.add_card(card)
                return f"❌ Invalid target: {who}"
    
    def clear_hands(self) -> str:
        """Move all cards from player and dealer hands to discard pile (keeps players)"""
        total_cleared = 0
        
        for player_num, player_zone in self.players.items():
            player_cards = player_zone.clear()
            for card in player_cards:
                self.discard.add_card(card)
            total_cleared += len(player_cards)
        
        dealer_cards = self.dealer_hand.clear()
        for card in dealer_cards:
            self.discard.add_card(card)
        
        face_down_count = self.face_down.total_cards()
        if face_down_count > 0:
            self.face_down.clear()
            for _ in range(face_down_count):
                self.discard.add_card("UNKNOWN")
        
        self.face_down_owners.clear()
        
        total = total_cleared + len(dealer_cards) + face_down_count
        player_count = len(self.players)
        return f"🗑️ Cleared all hands: {total_cleared} from players, {len(dealer_cards)} from dealer, {face_down_count} face down → Discard pile\n({player_count} player(s) still at table)"
    
    def shuffle(self):
        """Return all cards to deck"""
        for player_zone in self.players.values():
            player_zone.clear()
        
        self.dealer_hand.clear()
        self.discard.clear()
        self.face_down.clear()
        self.face_down_owners.clear()
        self.players.clear()
        self.next_player_number = 1
        
        self.initialize_deck()
        return "🔀 Shuffled! All cards returned to deck. All players removed."
    
    def get_status(self) -> str:
        """Get complete game state"""
        face_down_status = ""
        if self.face_down.total_cards() > 0:
            fd_list = []
            for owner_key, count in self.face_down_owners.items():
                if owner_key == "dealer":
                    fd_list.append(f"Dealer: {count}×🂠")
                else:
                    player_num = owner_key.replace("player", "")
                    fd_list.append(f"Player {player_num}: {count}×🂠")
            face_down_status = f"🂠 Face Down ({self.face_down.total_cards()}): {', '.join(fd_list)}\n"
        
        deck_and_facedown = self.deck.total_cards() + self.face_down.total_cards()
        
        player_hands = ""
        if self.players:
            for player_num in sorted(self.players.keys()):
                player_hands += f"👤 {self.players[player_num]}\n"
        else:
            player_hands = "👤 No players added yet\n"
        
        total_in_play = self.deck.total_cards() + self.face_down.total_cards() + self.dealer_hand.total_cards() + self.discard.total_cards()
        for player_zone in self.players.values():
            total_in_play += player_zone.total_cards()
        
        return (
            f"**Game State**\n\n"
            f"🎴 Deck + Face Down ({deck_and_facedown} cards)\n"
            f"   ↳ {self.deck}\n"
            f"   ↳ {face_down_status.strip() if face_down_status else 'Face Down: None'}\n"
            f"{player_hands}"
            f"🎰 {self.dealer_hand}\n"
            f"🗑️ {self.discard}\n\n"
            f"Total cards tracked: {total_in_play}/52"
        )
    
    def calculate_hand_total(self, cards: List[str]) -> int:
        """Calculate the total value of a hand (best non-bust value)"""
        total = 0
        aces = 0
        
        for card in cards:
            card = card.upper().strip()
            if card in ['J', 'Q', 'K']:
                total += 10
            elif card == 'A':
                aces += 1
                total += 11
            else:
                total += int(card)
        
        # Adjust for aces
        while total > 21 and aces > 0:
            total -= 10
            aces -= 1
        
        return total
    
    def get_hand_cards(self, who: str) -> List[str]:
        """Get the cards in a specific player's or dealer's hand"""
        who_lower = who.lower()
        
        if who_lower == "dealer":
            return self.dealer_hand.get_all_cards()
        else:
            player_num, player_zone = self.get_player_zone(who)
            if player_zone:
                return player_zone.get_all_cards()
            else:
                return []
    
    def calculate_probability_after_hit(self, who: str, threshold: int) -> str:
        """Calculate probability of hand being >, <, or = threshold after hitting"""
        current_hand = self.get_hand_cards(who)
        
        if not current_hand:
            return f"❌ {who} has no cards in hand"
        
        current_total = self.calculate_hand_total(current_hand)
        
        # Get available cards in deck (excluding face down)
        available_cards = dict(self.deck.cards)
        total_available = sum(available_cards.values())
        
        if total_available == 0:
            return "❌ No cards left in deck to draw"
        
        # Calculate outcomes for each possible card
        greater_count = 0
        less_count = 0
        equal_count = 0
        
        for card, count in available_cards.items():
            # Simulate adding this card
            test_hand = current_hand + [card]
            new_total = self.calculate_hand_total(test_hand)
            
            if new_total > threshold:
                greater_count += count
            elif new_total < threshold:
                less_count += count
            else:
                equal_count += count
        
        # Calculate percentages
        greater_pct = (greater_count / total_available) * 100
        less_pct = (less_count / total_available) * 100
        equal_pct = (equal_count / total_available) * 100
        
        who_display = "Dealer" if who.lower() == "dealer" else f"Player {self.get_player_zone(who)[0]}"
        
        return (
            f"**Probability Analysis: {who_display}**\n\n"
            f"Current hand: {', '.join(current_hand)} (Total: {current_total})\n"
            f"Threshold: {threshold}\n"
            f"Available cards in deck: {total_available}\n\n"
            f"After hitting:\n"
            f"• Greater than {threshold}: {greater_pct:.1f}% ({greater_count}/{total_available} cards)\n"
            f"• Less than {threshold}: {less_pct:.1f}% ({less_count}/{total_available} cards)\n"
            f"• Equal to {threshold}: {equal_pct:.1f}% ({equal_count}/{total_available} cards)"
        )

# ============================================================================
# AGENT PROTOCOL HANDLERS WITH PERSISTENT STORAGE
# ============================================================================

# Global game state that persists across messages during agent runtime
GLOBAL_GAME_STATE = None

def get_or_create_game_state() -> GameState:
    """Get existing game state or create new one"""
    global GLOBAL_GAME_STATE
    if GLOBAL_GAME_STATE is None:
        GLOBAL_GAME_STATE = GameState()
    return GLOBAL_GAME_STATE

def reset_game_state():
    """Reset the global game state"""
    global GLOBAL_GAME_STATE
    GLOBAL_GAME_STATE = GameState()

blackjack_protocol = Protocol(name="BlackjackAdvisor", version="1.0")

async def load_game_state(ctx: Context) -> GameState:
    """Load game state - first try global, then storage, then create new"""
    global GLOBAL_GAME_STATE
    
    # Use global state if it exists
    if GLOBAL_GAME_STATE is not None:
        ctx.logger.info("[STATE] Using existing global game state")
        return GLOBAL_GAME_STATE
    
    # Try loading from storage (synchronous, not async)
    try:
        state_json = ctx.storage.get("game_state")
        if state_json:
            ctx.logger.info("[STATE] Loading game state from storage")
            GLOBAL_GAME_STATE = GameState.from_dict(json.loads(state_json))
            return GLOBAL_GAME_STATE
    except Exception as e:
        ctx.logger.warning(f"[STATE] Could not load from storage: {e}")
    
    # Create new state
    ctx.logger.info("[STATE] Creating new game state")
    GLOBAL_GAME_STATE = GameState()
    return GLOBAL_GAME_STATE

async def save_game_state(ctx: Context, game_state: GameState):
    """Save game state to both global and storage"""
    global GLOBAL_GAME_STATE
    GLOBAL_GAME_STATE = game_state
    
    try:
        state_json = json.dumps(game_state.to_dict())
        ctx.storage.set("game_state", state_json)
        ctx.logger.info("[STATE] Game state saved to storage")
    except Exception as e:
        ctx.logger.error(f"[STATE] Failed to save to storage: {e}")

@blackjack_protocol.on_message(model=StartSessionContent)
async def handle_session_start(ctx: Context, sender: str, msg: StartSessionContent):
    ctx.logger.info(f"[SESSION START] Received from {sender}")
    welcome = (
        "🃏 **Blackjack Card Tracker Ready!**\n\n"
        "Commands:\n"
        "• `Add Player` - Add a new player (auto-numbered)\n"
        "• `Deal <card> to Player 1/Dealer` - Deal initial cards\n"
        "• `Deal Face Down to Player 2/Dealer` - Deal unknown card\n"
        "• `Flip Player 1/Dealer <card>` - Reveal face down card\n"
        "• `Player 1 Hit <card>` - Player draws a card\n"
        "• `Dealer Hit <card>` - Dealer draws a card\n"
        "• `Probability of <number> after Player <#>/Dealer hit` - Calculate odds\n"
        "• `Clear Hands` - Move all hands to discard\n"
        "• `Status` - See all card locations\n"
        "• `Shuffle` - Reset deck and remove all players\n\n"
        "Example: `Probability of 21 after Player 1 hit`"
    )
    await ctx.send(sender, ChatMessage(content=[TextContent(text=welcome)]))

@blackjack_protocol.on_message(model=ChatMessage)
async def handle_chat_message(ctx: Context, sender: str, msg: ChatMessage):
    ctx.logger.info(f"[CHAT] Received ChatMessage from {sender}")
    
    # Load game state from storage
    game_state = await load_game_state(ctx)
    ctx.logger.info(f"[DEBUG] Game state loaded - Players: {list(game_state.players.keys())}, Deck cards: {game_state.deck.total_cards()}, Next player #: {game_state.next_player_number}")
    
    query = ""
    try:
        if isinstance(msg.content, list):
            for item in msg.content:
                if hasattr(item, 'text'):
                    query += str(item.text)
        elif hasattr(msg.content, 'text'):
            query = str(msg.content.text)
        else:
            query = str(msg.content)
    except Exception as e:
        ctx.logger.error(f"[CHAT] Content extraction failed: {e}")
        await ctx.send(sender, ChatMessage(content=[TextContent(text=f"Error: {e}")]))
        return
    
    if not query or not query.strip():
        return
    
    query = query.strip()
    query_lower = query.lower()
    ctx.logger.info(f"[CHAT] Processing: {query}")
    
    response = ""
    
    if query_lower in ["add player", "new player", "add"]:
        response = game_state.add_player()
        response += "\n\n" + game_state.get_status()
    
    elif query_lower in ["status", "state", "show"]:
        response = game_state.get_status()
    
    elif query_lower in ["shuffle", "reset", "new deck"]:
        response = game_state.shuffle()
    
    elif query_lower in ["clear hands", "clear hand", "clear"]:
        response = game_state.clear_hands()
        response += "\n\n" + game_state.get_status()
    
    elif query_lower.startswith("deal"):
        try:
            if "face down" in query_lower:
                parts = query.lower().split("to")
                if len(parts) == 2:
                    target = parts[1].strip()
                    response = game_state.deal_face_down(target)
                    response += "\n\n" + game_state.get_status()
                else:
                    response = "Format: `Deal Face Down to Player 1/Dealer`"
            else:
                parts = query.split()
                if len(parts) >= 4 and parts[2].lower() == "to":
                    card = parts[1].upper()
                    target = parts[3]
                    response = game_state.deal_card(card, target)
                    response += "\n\n" + game_state.get_status()
                else:
                    response = "Format: `Deal <card> to Player 1/Dealer`\nExample: `Deal A to Player 1`"
        except Exception as e:
            response = f"Error parsing deal command: {e}"
    
    elif query_lower.startswith("flip"):
        try:
            parts = query.split()
            if len(parts) >= 3:
                who = parts[1]
                card = parts[2].upper()
                response = game_state.flip_card(who, card)
                response += "\n\n" + game_state.get_status()
            else:
                response = "Format: `Flip Player 1/Dealer <card>`\nExample: `Flip Dealer K`"
        except Exception as e:
            response = f"Error: {e}"
    
    elif query_lower.startswith("player") and "hit" in query_lower:
        try:
            parts = query.split()
            hit_index = -1
            for i, part in enumerate(parts):
                if part.lower() == "hit":
                    hit_index = i
                    break
            
            if hit_index >= 0 and len(parts) > hit_index + 1:
                card = parts[hit_index + 1].upper()
                player_id = " ".join(parts[1:hit_index]) if hit_index > 1 else "1"
                response = game_state.hit(card, player_id)
                response += "\n\n" + game_state.get_status()
            else:
                response = "Format: `Player 1 Hit <card>`\nExample: `Player 1 Hit 5`"
        except Exception as e:
            response = f"Error: {e}"
    
    elif query_lower.startswith("dealer hit"):
        try:
            parts = query.split()
            if len(parts) >= 3:
                card = parts[2].upper()
                response = game_state.hit(card, "dealer")
                response += "\n\n" + game_state.get_status()
            else:
                response = "Format: `Dealer Hit <card>`\nExample: `Dealer Hit K`"
        except Exception as e:
            response = f"Error: {e}"
    
    elif "probability" in query_lower and "after" in query_lower and "hit" in query_lower:
        # Format: "Probability of 21 after Player 1 hit" or "Probability of 17 after Dealer hit"
        try:
            # Extract threshold number
            import re
            threshold_match = re.search(r'of\s+(\d+)', query_lower)
            
            # Extract who (player number or dealer)
            who = None
            if "dealer" in query_lower:
                who = "dealer"
            else:
                player_match = re.search(r'player\s*(\d+)', query_lower)
                if player_match:
                    who = player_match.group(1)
            
            if threshold_match and who:
                threshold = int(threshold_match.group(1))
                response = game_state.calculate_probability_after_hit(who, threshold)
            else:
                response = "Format: `Probability of <number> after Player <#>/Dealer hit`\nExample: `Probability of 21 after Player 1 hit`"
        except Exception as e:
            response = f"Error: {e}"
    
    elif query_lower.startswith("dealer hit"):
        try:
            parts = query.split()
            if len(parts) >= 3:
                card = parts[2].upper()
                response = game_state.hit(card, "dealer")
                response += "\n\n" + game_state.get_status()
            else:
                response = "Format: `Dealer Hit <card>`\nExample: `Dealer Hit K`"
        except Exception as e:
            response = f"Error: {e}"
    
    else:
        response = (
            "Unknown command. Available commands:\n"
            "• `Add Player`\n"
            "• `Deal <card> to Player 1/Dealer`\n"
            "• `Deal Face Down to Player 1/Dealer`\n"
            "• `Flip Player 1/Dealer <card>`\n"
            "• `Player 1 Hit <card>`\n"
            "• `Dealer Hit <card>`\n"
            "• `Probability of <number> after Player <#>/Dealer hit`\n"
            "• `Clear Hands`\n"
            "• `Status`\n"
            "• `Shuffle`"
        )
    
    # Save game state after processing
    await save_game_state(ctx, game_state)
    ctx.logger.info(f"[DEBUG] Game state saved - Players: {list(game_state.players.keys())}, Deck cards: {game_state.deck.total_cards()}, Next player #: {game_state.next_player_number}")
    
    ctx.logger.info(f"[CHAT] Sending response")
    await ctx.send(sender, ChatMessage(content=[TextContent(text=response)]))

blackjack_agent = Agent(
    name="BlackjackPredictor",
    seed="blackjack_probability_predictor_seed",
)
blackjack_agent.include(blackjack_protocol)

if __name__ == "__main__":
    blackjack_agent.run()
