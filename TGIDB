import telegram
import mysql.connector
from dotenv import load_dotenv
import os

# Load the database credentials from the .env file
load_dotenv()
database_name = os.getenv('DATABASE_NAME')
database_host = os.getenv('DATABASE_HOST')
database_user = os.getenv('DATABASE_USER')
database_password = os.getenv('DATABASE_PASSWORD')

# Replace YOUR_API_TOKEN with your Telegram bot API token
bot = telegram.Bot(token='BOTTOKEN')

# Define the Telegram ID number that is allowed to use the bot
ALLOWED_ID = 121456602

# Create a connection to the MySQL database
conn = mysql.connector.connect(
    host=database_host,
    user=database_user,
    password=database_password,
    database=database_name
)

# Create a cursor object to execute SQL queries
cursor = conn.cursor()

# Create a table to store the gathered member IDs
def create_members_table(table_name):
    cursor.execute(f"CREATE TABLE IF NOT EXISTS {table_name} (id INT AUTO_INCREMENT PRIMARY KEY, member_id INT)")

# Add a member ID to the specified table
def add_member_to_table(table_name, member_id):
    cursor.execute(f"INSERT INTO {table_name} (member_id) VALUES (%s)", (member_id,))
    conn.commit()

# Find groups in a specific subject and join them
def find_and_join_groups(subject):
    # Replace YOUR_CHAT_ID with your chat ID
    chat_id = 'YOUR_CHAT_ID'

    # Search for groups in the specified subject
    groups = bot.search_chat_messages(chat_id=chat_id, query=subject, filter='supergroup')

    # Join each group and gather the member IDs
    for group in groups:
        group_id = group.chat.id
        bot.join_chat(chat_id=group_id)
        members = bot.get_chat_members(chat_id=group_id)
        table_name = f"{subject}_{group_id}"
        create_members_table(table_name)
        for member in members:
            member_id = member.user.id
            add_member_to_table(table_name, member_id)

# Handle user input
def handle_user_input(update, context):
    # Get the user's input
    user_input = update.message.text

    # Find groups in the specified subject and join them
    find_and_join_groups(user_input)

    # Send a message to the user to confirm the search and join
    message_text = f'Groups in the subject "{user_input}" have been found and joined.'
    bot.send_message(chat_id=update.message.chat_id, text=message_text)

# Show a list of tables and their member count
def show_tables(update, context):
    cursor.execute("SHOW TABLES")
    tables = cursor.fetchall()
    message_text = ''
    for table in tables:
        table_name = table[0]
        cursor.execute(f"SELECT COUNT(*) FROM {table_name}")
        member_count = cursor.fetchone()[0]
        message_text += f"{table_name}: {member_count} members\n"
    bot.send_message(chat_id=update.message.chat_id, text=message_text)

# Define the conversation handler
from telegram.ext import ConversationHandler
conversation_handler = ConversationHandler(
    entry_points=[telegram.ext.CommandHandler('start', handle_user_input)],
    states={},
    fallbacks=[]
)

# Add the conversation handler to the dispatcher
from telegram.ext import Updater
updater = Updater(token='YOUR_API_TOKEN', use_context=True)
dispatcher = updater.dispatcher
dispatcher.add_handler(conversation_handler)

# Define the menu buttons
from telegram import InlineKeyboardButton, InlineKeyboardMarkup
def menu_buttons(update, context):
    keyboard = [
        [InlineKeyboardButton("New Subject", callback_data='new_subject')],
        [InlineKeyboardButton("Add to", callback_data='add_to')],
        [InlineKeyboardButton("Tables", callback_data='tables')]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    update.message.reply_text('Please choose an option:', reply_markup=reply_markup)

# Handle menu button callbacks
def menu_button_callback(update, context):
    query = update.callback_query
    if query.data == 'new_subject':
        bot.send_message(chat_id=query.message.chat_id, text='Please enter a new subject:')
    elif query.data == 'add_to':
        bot.send_message(chat_id=query.message.chat_id, text='Please enter the name of the table you want to add to:')
        context.user_data['add_to'] = True
    elif query.data == 'tables':
        show_tables(update, context)

# Define the callback query handler
from telegram.ext import CallbackQueryHandler
callback_query_handler = CallbackQueryHandler(menu_button_callback)
dispatcher.add_handler(callback_query_handler)

# Define the message handler for adding members to a table
def add_to_table(update, context):
    if 'add_to' in context.user_data:
        table_name = update.message.text
        context.user_data['table_name'] = table_name
        bot.send_message(chat_id=update.message.chat_id, text=f'Please enter the member IDs you want to add to {table_name}, separated by commas:')
        return 'add_members'
    else:
        return ConversationHandler.END

# Define the message handler for adding members to a table
def add_members(update, context):
    if 'table_name' in context.user_data:
        table_name = context.user_data['table_name']
        member_ids = update.message.text.split(',')
        for member_id in member_ids:
            add_member_to_table(table_name, member_id)
        bot.send_message(chat_id=update.message.chat_id, text=f'{len(member_ids)} members have been added to {table_name}.')
        del context.user_data['add_to']
        del context.user_data['table_name']
        return ConversationHandler.END
    else:
        return ConversationHandler.END

# Define the conversation handler for adding members to a table
add_members_handler = ConversationHandler(
    entry_points=[telegram.ext.MessageHandler(telegram.ext.Filters.text & ~telegram.ext.Filters.command, add_to_table)],
    states={
        'add_members': [telegram.ext.MessageHandler(telegram.ext.Filters.text & ~telegram.ext.Filters.command, add_members)]
    },
    fallbacks=[]
)
dispatcher.add_handler(add_members_handler)

# Start the bot
updater.start_polling()

# Show the menu buttons to the user
dispatcher.add_handler(telegram.ext.CommandHandler('menu', menu_buttons))
