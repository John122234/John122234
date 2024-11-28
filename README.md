import time
from telegram import Update
from telegram.ext import Updater, CommandHandler, CallbackContext
from web3 import Web3

# Blockchain Configuration
WEB3_PROVIDER = "https://bsc-dataseed.binance.org/"  # Binance Smart Chain Mainnet
PRIVATE_KEY = "YOUR_PRIVATE_KEY"
WALLET_ADDRESS = "YOUR_WALLET_ADDRESS"
TOKEN_ADDRESS = "YOUR_TOKEN_CONTRACT_ADDRESS"
TOKEN_ABI = [...]  # Replace with your token's ABI

# Connect to blockchain
web3 = Web3(Web3.HTTPProvider(WEB3_PROVIDER))
token_contract = web3.eth.contract(address=TOKEN_ADDRESS, abi=TOKEN_ABI)

# User farming data
user_data = {}

# Farming Configuration
POINTS_PER_HOUR = 10  # Points earned per hour
REFERRAL_POINTS = 50  # Points for each successful referral
TASK_POINTS = 20  # Points for completing a task (e.g., joining a channel)

# Command: /start
def start(update: Update, context: CallbackContext):
    update.message.reply_text(
        "Welcome to the Farming Bot! 🌾\n\n"
        "Use /farm to start farming points.\n"
        "Use /points to check your points.\n"
        "Use /register_wallet to connect your wallet.\n"
        "Use /task to complete tasks for points.\n"
        "Use /refer to get your referral link."
    )

# Command: /farm
def start_farming(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    current_time = time.time()

    if user_id not in user_data:
        user_data[user_id] = {"points": 0, "last_farmed": current_time, "wallet": None, "referred_by": None}
        update.message.reply_text("Farming started! 🌱 Keep earning points over time.")
    else:
        update.message.reply_text("You are already farming! 🌾 Keep going!")

# Command: /register_wallet
def register_wallet(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    if len(context.args) == 0:
        update.message.reply_text("Please provide your wallet address. Example: /register_wallet 0xYourWalletAddress")
        return

    wallet_address = context.args[0]
    if not web3.isAddress(wallet_address):
        update.message.reply_text("Invalid wallet address. Please try again.")
        return

    if user_id not in user_data:
        user_data[user_id] = {"points": 0, "last_farmed": time.time(), "wallet": wallet_address, "referred_by": None}
        update.message.reply_text(f"Your wallet address {wallet_address} has been registered! ✅")
    else:
        user_data[user_id]["wallet"] = wallet_address
        update.message.reply_text(f"Wallet address updated to {wallet_address}.")

# Command: /points
def check_points(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    current_time = time.time()

    if user_id not in user_data:
        update.message.reply_text("You haven't started farming yet. Use /farm to start.")
        return

    # Calculate new points
    last_farmed = user_data[user_id]["last_farmed"]
    elapsed_hours = (current_time - last_farmed) / 3600
    new_points = int(elapsed_hours * POINTS_PER_HOUR)

    # Update points and last farmed time
    user_data[user_id]["points"] += new_points
    user_data[user_id]["last_farmed"] = current_time

    update.message.reply_text(f"You have {user_data[user_id]['points']} points. 🌟")

# Command: /task
def complete_task(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id

    if user_id not in user_data:
        update.message.reply_text("You haven't started farming yet. Use /farm to start.")
        return

    # Add task points (e.g., join a Telegram channel)
    user_data[user_id]["points"] += TASK_POINTS
    update.message.reply_text(f"Task completed! 🎉 You earned {TASK_POINTS} points. Total points: {user_data[user_id]['points']}.")

# Command: /refer
def referral_link(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    if user_id not in user_data:
        update.message.reply_text("You haven't started farming yet. Use /farm to start.")
        return

    # Generate referral link
    referral_link = f"https://t.me/{context.bot.username}?start={user_id}"
    update.message.reply_text(f"Your referral link: {referral_link}\nInvite others to earn points!")

# Command: /join_channel (For task validation)
def join_channel(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    if user_id not in user_data:
        update.message.reply_text("You haven't started farming yet. Use /farm to start.")
        return

    # This command is to simulate channel joining
    update.message.reply_text("Thanks for joining the channel! 🎉 You earned points.")

    # Add task points for joining the channel
    user_data[user_id]["points"] += TASK_POINTS
    update.message.reply_text(f"Points for task: {TASK_POINTS}. Your total points: {user_data[user_id]['points']}.")

# Command: /points_referrals
def check_referrals(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    if user_id not in user_data:
        update.message.reply_text("You haven't started farming yet. Use /farm to start.")
        return

    # Count referrals (we use the referred_by field)
    referrals = sum(1 for data in user_data.values() if data["referred_by"] == user_id)
    update.message.reply_text(f"You have {referrals} referrals. Earn {REFERRAL_POINTS} points for each successful referral!")

# Command: /points
def check_referrals(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    current_time = time.time()

    if user_id not in user_data:
        update.message.reply_text("You haven't started farming yet. Use /farm to start.")
        return

    # Calculate new points
    last_farmed = user_data[user_id]["last_farmed"]
    elapsed_hours = (current_time - last_farmed) / 3600
    new_points = int(elapsed_hours * POINTS_PER_HOUR)

    # Update points and last farmed time
    user_data[user_id]["points"] += new_points
    user_data[user_id]["last_farmed"] = current_time

    update.message.reply_text(f"You have {user_data[user_id]['points']} points. 🌟")

# Command: /allocate
def send_tokens(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    if user_id not in user_data:
        update.message.reply_text("You haven't started farming yet. Use /farm to start.")
        return

    # Check if the user has enough points for conversion
    total_points = user_data[user_id]["points"]
    if total_points > 0:
        wallet_address = user_data[user_id]["wallet"]
        try:
            # Send tokens based on the points farmed (just an example, adjust the logic as needed)
            tx_hash = send_tokens_to_wallet(wallet_address, total_points // 100)  # 100 points per token
            update.message.reply_text(f"Sent {total_points // 100} tokens to your wallet! Transaction hash: {tx_hash}")
        except Exception as e:
            update.message.reply_text(f"Error in sending tokens: {e}")
    else:
        update.message.reply_text("You don't have enough points to claim tokens. Keep farming! 🌾")

# Function: Send Tokens to Wallet
def send_tokens_to_wallet(to_address, amount):
    nonce = web3.eth.getTransactionCount(WALLET_ADDRESS)
    tx = token_contract.functions.transfer(
        to_address, web3.toWei(amount, 'ether')  # You can change this to 'gwei' or another unit if needed
    ).buildTransaction({
        'chainId': 56,  # Binance Smart Chain Mainnet (change if using other blockchains)
        'gas': 2000000,
        'gasPrice': web3.toWei('5', 'gwei'),
        'nonce': nonce,
    })

    signed_tx = web3.eth.account.sign_transaction(tx, private_key=PRIVATE_KEY)
    tx_hash = web3.eth.send_raw_transaction(signed_tx.rawTransaction)
    return web3.toHex(tx_hash)

# Main function
def main():
    TOKEN = "YOUR_TELEGRAM_BOT_TOKEN"
    updater = Updater(TOKEN, use_context=True)
    dp = updater.dispatcher

    # Command handlers
    dp.add_handler(CommandHandler("start", start))
    dp.add_handler(CommandHandler("farm", start_farming))
    dp.add_handlerimport time
from telegram import Update
from telegram.ext import Updater, CommandHandler, CallbackContext
from web3 import Web3

# Blockchain Configuration
WEB3_PROVIDER = "https://bsc-dataseed.binance.org/"  # Binance Smart Chain Mainnet
PRIVATE_KEY = "YOUR_PRIVATE_KEY"
WALLET_ADDRESS = "YOUR_WALLET_ADDRESS"
TOKEN_ADDRESS = "YOUR_TOKEN_CONTRACT_ADDRESS"
TOKEN_ABI = [...]  # Replace with your token's ABI

# Connect to blockchain
web3 = Web3(Web3.HTTPProvider(WEB3_PROVIDER))
token_contract = web3.eth.contract(address=TOKEN_ADDRESS, abi=TOKEN_ABI)

# User farming data
user_data = {}

# Farming Configuration
POINTS_PER_HOUR = 10  # Points earned per hour
REFERRAL_POINTS = 50  # Points for each successful referral
TASK_POINTS = 20  # Points for completing a task (e.g., joining a channel)

# Command: /start
def start(update: Update, context: CallbackContext):
    update.message.reply_text(
        "Welcome to the Farming Bot! 🌾\n\n"
        "Use /farm to start farming points.\n"
        "Use /points to check your points.\n"
        "Use /register_wallet to connect your wallet.\n"
        "Use /task to complete tasks for points.\n"
        "Use /refer to get your referral link."
    )

# Command: /farm
def start_farming(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    current_time = time.time()

    if user_id not in user_data:
        user_data[user_id] = {"points": 0, "last_farmed": current_time, "wallet": None, "referred_by": None}
        update.message.reply_text("Farming started! 🌱 Keep earning points over time.")
    else:
        update.message.reply_text("You are already farming! 🌾 Keep going!")

# Command: /register_wallet
def register_wallet(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    if len(context.args) == 0:
        update.message.reply_text("Please provide your wallet address. Example: /register_wallet 0xYourWalletAddress")
        return

    wallet_address = context.args[0]
    if not web3.isAddress(wallet_address):
        update.message.reply_text("Invalid wallet address. Please try again.")
        return

    if user_id not in user_data:
        user_data[user_id] = {"points": 0, "last_farmed": time.time(), "wallet": wallet_address, "referred_by": None}
        update.message.reply_text(f"Your wallet address {wallet_address} has been registered! ✅")
    else:
        user_data[user_id]["wallet"] = wallet_address
        update.message.reply_text(f"Wallet address updated to {wallet_address}.")

# Command: /points
def check_points(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    current_time = time.time()

    if user_id not in user_data:
        update.message.reply_text("You haven't started farming yet. Use /farm to start.")
        return

    # Calculate new points
    last_farmed = user_data[user_id]["last_farmed"]
    elapsed_hours = (current_time - last_farmed) / 3600
    new_points = int(elapsed_hours * POINTS_PER_HOUR)

    # Update points and last farmed time
    user_data[user_id]["points"] += new_points
    user_data[user_id]["last_farmed"] = current_time

    update.message.reply_text(f"You have {user_data[user_id]['points']} points. 🌟")

# Command: /task
def complete_task(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id

    if user_id not in user_data:
        update.message.reply_text("You haven't started farming yet. Use /farm to start.")
        return

    # Add task points (e.g., join a Telegram channel)
    user_data[user_id]["points"] += TASK_POINTS
    update.message.reply_text(f"Task completed! 🎉 You earned {TASK_POINTS} points. Total points: {user_data[user_id]['points']}.")

# Command: /refer
def referral_link(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    if user_id not in user_data:
        update.message.reply_text("You haven't started farming yet. Use /farm to start.")
        return

    # Generate referral link
    referral_link = f"https://t.me/{context.bot.username}?start={user_id}"
    update.message.reply_text(f"Your referral link: {referral_link}\nInvite others to earn points!")

# Command: /join_channel (For task validation)
def join_channel(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    if user_id not in user_data:
        update.message.reply_text("You haven't started farming yet. Use /farm to start.")
        return

    # This command is to simulate channel joining
    update.message.reply_text("Thanks for joining the channel! 🎉 You earned points.")

    # Add task points for joining the channel
    user_data[user_id]["points"] += TASK_POINTS
    update.message.reply_text(f"Points for task: {TASK_POINTS}. Your total points: {user_data[user_id]['points']}.")

# Command: /points_referrals
def check_referrals(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    if user_id not in user_data:
        update.message.reply_text("You haven't started farming yet. Use /farm to start.")
        return

    # Count referrals (we use the referred_by field)
    referrals = sum(1 for data in user_data.values() if data["referred_by"] == user_id)
    update.message.reply_text(f"You have {referrals} referrals. Earn {REFERRAL_POINTS} points for each successful referral!")

# Command: /points
def check_referrals(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    current_time = time.time()

    if user_id not in user_data:
        update.message.reply_text("You haven't started farming yet. Use /farm to start.")
        return

    # Calculate new points
    last_farmed = user_data[user_id]["last_farmed"]
    elapsed_hours = (current_time - last_farmed) / 3600
    new_points = int(elapsed_hours * POINTS_PER_HOUR)

    # Update points and last farmed time
    user_data[user_id]["points"] += new_points
    user_data[user_id]["last_farmed"] = current_time

    update.message.reply_text(f"You have {user_data[user_id]['points']} points. 🌟")

# Command: /allocate
def send_tokens(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    if user_id not in user_data:
        update.message.reply_text("You haven't started farming yet. Use /farm to start.")
        return

    # Check if the user has enough points for conversion
    total_points = user_data[user_id]["points"]
    if total_points > 0:
        wallet_address = user_data[user_id]["wallet"]
        try:
            # Send tokens based on the points farmed (just an example, adjust the logic as needed)
            tx_hash = send_tokens_to_wallet(wallet_address, total_points // 100)  # 100 points per token
            update.message.reply_text(f"Sent {total_points // 100} tokens to your wallet! Transaction hash: {tx_hash}")
        except Exception as e:
            update.message.reply_text(f"Error in sending tokens: {e}")
    else:
        update.message.reply_text("You don't have enough points to claim tokens. Keep farming! 🌾")

# Function: Send Tokens to Wallet
def send_tokens_to_wallet(to_address, amount):
    nonce = web3.eth.getTransactionCount(WALLET_ADDRESS)
    tx = token_contract.functions.transfer(
        to_address, web3.toWei(amount, 'ether')  # You can change this to 'gwei' or another unit if needed
    ).buildTransaction({
        'chainId': 56,  # Binance Smart Chain Mainnet (change if using other blockchains)
        'gas': 2000000,
        'gasPrice': web3.toWei('5', 'gwei'),
        'nonce': nonce,
    })

    signed_tx = web3.eth.account.sign_transaction(tx, private_key=PRIVATE_KEY)
    tx_hash = web3.eth.send_raw_transaction(signed_tx.rawTransaction)
    return web3.toHex(tx_hash)

# Main function
def main():
    TOKEN = "YOUR_TELEGRAM_BOT_TOKEN"
    updater = Updater(TOKEN, use_context=True)
    dp = updater.dispatcher

    # Command handlers
    dp.add_handler(CommandHandler("start", start))
    dp.add_handler(CommandHandler("farm", start_farming))
    dp.add_handlerimport time
from telegram import Update
from telegram.ext import Updater, CommandHandler, CallbackContext
from web3 import Web3

# Blockchain Configuration
WEB3_PROVIDER = "https://bsc-dataseed.binance.org/"  # Binance Smart Chain Mainnet
PRIVATE_KEY = "YOUR_PRIVATE_KEY"
WALLET_ADDRESS = "YOUR_WALLET_ADDRESS"
TOKEN_ADDRESS = "YOUR_TOKEN_CONTRACT_ADDRESS"
TOKEN_ABI = [...]  # Replace with your token's ABI

# Connect to blockchain
web3 = Web3(Web3.HTTPProvider(WEB3_PROVIDER))
token_contract = web3.eth.contract(address=TOKEN_ADDRESS, abi=TOKEN_ABI)

# User farming data
user_data = {}

# Farming Configuration
POINTS_PER_HOUR = 10  # Points earned per hour
REFERRAL_POINTS = 50  # Points for each successful referral
TASK_POINTS = 20  # Points for completing a task (e.g., joining a channel)

# Command: /start
def start(update: Update, context: CallbackContext):
    update.message.reply_text(
        "Welcome to the Farming Bot! 🌾\n\n"
        "Use /farm to start farming points.\n"
        "Use /points to check your points.\n"
        "Use /register_wallet to connect your wallet.\n"
        "Use /task to complete tasks for points.\n"
        "Use /refer to get your referral link."
    )

# Command: /farm
def start_farming(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    current_time = time.time()

    if user_id not in user_data:
        user_data[user_id] = {"points": 0, "last_farmed": current_time, "wallet": None, "referred_by": None}
        update.message.reply_text("Farming started! 🌱 Keep earning points over time.")
    else:
        update.message.reply_text("You are already farming! 🌾 Keep going!")

# Command: /register_wallet
def register_wallet(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    if len(context.args) == 0:
        update.message.reply_text("Please provide your wallet address. Example: /register_wallet 0xYourWalletAddress")
        return

    wallet_address = context.args[0]
    if not web3.isAddress(wallet_address):
        update.message.reply_text("Invalid wallet address. Please try again.")
        return

    if user_id not in user_data:
        user_data[user_id] = {"points": 0, "last_farmed": time.time(), "wallet": wallet_address, "referred_by": None}
        update.message.reply_text(f"Your wallet address {wallet_address} has been registered! ✅")
    else:
        user_data[user_id]["wallet"] = wallet_address
        update.message.reply_text(f"Wallet address updated to {wallet_address}.")

# Command: /points
def check_points(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    current_time = time.time()

    if user_id not in user_data:
        update.message.reply_text("You haven't started farming yet. Use /farm to start.")
        return

    # Calculate new points
    last_farmed = user_data[user_id]["last_farmed"]
    elapsed_hours = (current_time - last_farmed) / 3600
    new_points = int(elapsed_hours * POINTS_PER_HOUR)

    # Update points and last farmed time
    user_data[user_id]["points"] += new_points
    user_data[user_id]["last_farmed"] = current_time

    update.message.reply_text(f"You have {user_data[user_id]['points']} points. 🌟")

# Command: /task
def complete_task(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id

    if user_id not in user_data:
        update.message.reply_text("You haven't started farming yet. Use /farm to start.")
        return

    # Add task points (e.g., join a Telegram channel)
    user_data[user_id]["points"] += TASK_POINTS
    update.message.reply_text(f"Task completed! 🎉 You earned {TASK_POINTS} points. Total points: {user_data[user_id]['points']}.")

# Command: /refer
def referral_link(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    if user_id not in user_data:
        update.message.reply_text("You haven't started farming yet. Use /farm to start.")
        return

    # Generate referral link
    referral_link = f"https://t.me/{context.bot.username}?start={user_id}"
    update.message.reply_text(f"Your referral link: {referral_link}\nInvite others to earn points!")

# Command: /join_channel (For task validation)
def join_channel(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    if user_id not in user_data:
        update.message.reply_text("You haven't started farming yet. Use /farm to start.")
        return

    # This command is to simulate channel joining
    update.message.reply_text("Thanks for joining the channel! 🎉 You earned points.")

    # Add task points for joining the channel
    user_data[user_id]["points"] += TASK_POINTS
    update.message.reply_text(f"Points for task: {TASK_POINTS}. Your total points: {user_data[user_id]['points']}.")

# Command: /points_referrals
def check_referrals(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    if user_id not in user_data:
        update.message.reply_text("You haven't started farming yet. Use /farm to start.")
        return

    # Count referrals (we use the referred_by field)
    referrals = sum(1 for data in user_data.values() if data["referred_by"] == user_id)
    update.message.reply_text(f"You have {referrals} referrals. Earn {REFERRAL_POINTS} points for each successful referral!")

# Command: /points
def check_referrals(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    current_time = time.time()

    if user_id not in user_data:
        update.message.reply_text("You haven't started farming yet. Use /farm to start.")
        return

    # Calculate new points
    last_farmed = user_data[user_id]["last_farmed"]
    elapsed_hours = (current_time - last_farmed) / 3600
    new_points = int(elapsed_hours * POINTS_PER_HOUR)

    # Update points and last farmed time
    user_data[user_id]["points"] += new_points
    user_data[user_id]["last_farmed"] = current_time

    update.message.reply_text(f"You have {user_data[user_id]['points']} points. 🌟")

# Command: /allocate
def send_tokens(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    if user_id not in user_data:
        update.message.reply_text("You haven't started farming yet. Use /farm to start.")
        return

    # Check if the user has enough points for conversion
    total_points = user_data[user_id]["points"]
    if total_points > 0:
        wallet_address = user_data[user_id]["wallet"]
        try:
            # Send tokens based on the points farmed (just an example, adjust the logic as needed)
            tx_hash = send_tokens_to_wallet(wallet_address, total_points // 100)  # 100 points per token
            update.message.reply_text(f"Sent {total_points // 100} tokens to your wallet! Transaction hash: {tx_hash}")
        except Exception as e:
            update.message.reply_text(f"Error in sending tokens: {e}")
    else:
        update.message.reply_text("You don't have enough points to claim tokens. Keep farming! 🌾")

# Function: Send Tokens to Wallet
def send_tokens_to_wallet(to_address, amount):
    nonce = web3.eth.getTransactionCount(WALLET_ADDRESS)
    tx = token_contract.functions.transfer(
        to_address, web3.toWei(amount, 'ether')  # You can change this to 'gwei' or another unit if needed
    ).buildTransaction({
        'chainId': 56,  # Binance Smart Chain Mainnet (change if using other blockchains)
        'gas': 2000000,
        'gasPrice': web3.toWei('5', 'gwei'),
        'nonce': nonce,
    })

    signed_tx = web3.eth.account.sign_transaction(tx, private_key=PRIVATE_KEY)
    tx_hash = web3.eth.send_raw_transaction(signed_tx.rawTransaction)
    return web3.toHex(tx_hash)

# Main function
def main():
    TOKEN = "YOUR_TELEGRAM_BOT_TOKEN"
    updater = Updater(TOKEN, use_context=True)
    dp = updater.dispatcher

    # Command handlers
    dp.add_handler(CommandHandler("start", start))
    dp.add_handler(CommandHandler("farm", start_farming))
    dp.add_handler
