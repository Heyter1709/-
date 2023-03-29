import discord
from discord.ext import commands
import random
import openai
from discord.ext.commands import has_permissions
import asyncio

openai.api_key = "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"
model_engine = "text-davinci-003"


intents = discord.Intents.all()
bot = commands.Bot(command_prefix="!", intents=intents)

role_ids = [1075842528722550804, 1076369327399374849]

conversation_history = {}

async def generate_response(question, speaker=None, topic=None):
    prompt = f"Q: {question}"
    if speaker:
        prompt += f"\nS: {speaker}"
    if topic:
        prompt += f"\nT: {topic}"
    prompt += "\nA:"
    response = openai.Completion.create(
        engine=model_engine,
        prompt=prompt,
        max_tokens=2048,
        n=1,
        stop=None,
        temperature=0.9,
    )
    answer = response.choices[0].text.strip()
    return answer

@bot.command()
async def ask(ctx):
    await ctx.send("Ожидайте...")
    question = ctx.message.content.strip()
    if question in conversation_history:
        answer = conversation_history[question]["answer"]
    else:
        answer = await generate_response(question)
        conversation_history[question] = {"answer": answer, "follow_up": {}}

    response = f"```\n{answer}\n```"

    while True:
        # Send the response to the user
        message = await ctx.send(response)

        # Add the "Add follow-up question" reaction to the message
        await message.add_reaction('➕')

        # Define a function that checks for the reaction from the original user
        def check(reaction, user):
            return user == ctx.message.author and str(reaction.emoji) == '➕' and reaction.message.id == message.id

        try:
            # Wait for the user to add the "Add follow-up question" reaction
            reaction, user = await bot.wait_for('reaction_add', timeout=60.0, check=check)
        except asyncio.TimeoutError:
            # Remove the "Add follow-up question" reaction if the user didn't respond within 60 seconds
            await message.clear_reaction('➕')
            break
        else:
            # Remove the "Add follow-up question" reaction
            await message.clear_reaction('➕')

            # Ask the user for the follow-up question
            await ctx.send("Введите дополнительный вопрос:")

            try:
                # Wait for the user to send the follow-up question
                new_question = await bot.wait_for('message', timeout=60.0, check=lambda m: m.author == ctx.author and m.channel == ctx.channel)
            except asyncio.TimeoutError:
                await ctx.send("Превышено время ожидания.")
            else:
                # Generate the answer to the follow-up question
                new_answer = await generate_response(new_question.content.strip(), speaker=None, topic=question)

                # Add the follow-up question and answer to the conversation history
                if question in conversation_history:
                    conversation_history[question]["follow_up"][new_question.content.strip()] = new_answer
                else:
                    conversation_history[question] = {"answer": answer, "follow_up": {new_question.content.strip(): new_answer}}

                # Update the response with the new conversation history
                response = f"```\n{format_conversation_history(conversation_history[question])}\n```"


def format_conversation_history(conversation_history):
    # Format the main answer
    formatted_history = f"Бот: {conversation_history['answer']}\n"

    # Format the follow-up questions and answers
    for i, (follow_up_question, follow_up_answer) in enumerate(conversation_history['follow_up'].items()):
        formatted_history += f"\nТы-{i + 1}: {follow_up_question}\nБот-{i + 1}: {follow_up_answer}"

    return formatted_history

@bot.command()
async def rand(ctx,*,question):
    answers = ['Да', 'Нет', 'Больше да чем нет', 'Скорее нет', 'Иди умойся чушка ебаная и не пиши мне больше!']
    response = random.choice(answers)
    embed = discord.Embed(title='Вопрос:', description=question, color=0x00ff00)
    embed.add_field(name='Ответ:', value=response, inline=False)
    await ctx.send(embed=embed)


@bot.command()
async def com(ctx):
    # текст для вывода
    text = """
    Список команд:

    **!rand** - Да или Нет.

    **!ask** - Задайте любой вопрос боту.

    **!remuve** - снимает все роли (доступна только высшей администрации)

    **!add_role** - <id>-человека <id>-роли (доступна только высшей администрации)

    """

    # создаем embed и добавляем в него текст
    embed = discord.Embed(title="Команды Поркшеяна", description=text, color=0x00ff00)

    # отправляем embed в чат
    await ctx.send(embed=embed)


@bot.command()
@commands.has_any_role(*role_ids)
async def remuve(ctx, member_id: int):
    guild = ctx.guild
    member = guild.get_member(member_id)
    roles_to_remove = [role for role in member.roles if role.name != "@everyone"]
    await member.remove_roles(*roles_to_remove)
    await ctx.send(f"Роли сняты {member.display_name}.")


@bot.command()
@commands.has_any_role(*role_ids)
async def add_role(ctx, user_id: int, role_id: int):
    user = await bot.fetch_user(user_id)
    role = ctx.guild.get_role(role_id)
    await ctx.guild.get_member(user_id).add_roles(role, reason="Роль добавлена!")
    await ctx.send(f"Роль добавлена {role.name} юзеру {user.mention}.")



bot.run('MTA2NjM5MjEwOTYyNTYzOTA1NA.GIxlat.5OIWZMqrHE3saKXCjFFcbc90pF0wHRCh7IniSg')
