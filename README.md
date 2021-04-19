# chatBot

#Using telegram API
#This Bot will help us learn Aptitude based questions.
#It will also give suggestions for learning materials from where you can prepare.
#You can even take mock tests.
#Your data will be stored in a well maintained database.

import pymysql.cursors
import telebot
from telebot.types import InlineKeyboardMarkup, InlineKeyboardButton
import logging
from telebot import types
from random import randrange

API_TOKEN = '1226366316:AAEW850nlDMlDJhWcS00dcPU7605dCT5o1M'

bot = telebot.TeleBot(API_TOKEN, parse_mode='HTML')

SUBJECTS = ['ALLIGATION_OF_MIXTURE', 'PROBLEMS_ON_AGES','PROBLEMS_ON_TRAINS','LOGICAL_QUESTIONS','PROGRAMMING']


logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)

logger = logging.getLogger(__name__)


#user = update.message.from_user
#message.from_user.id in this user id will store


conn = pymysql.connect(host='127.0.0.1', user='root', password='', db='quiz', cursorclass=pymysql.cursors.DictCursor)
states= None
ans=''
c_op=''
ques_id,l=0,0
result=''
arr=[]

# Handle '/start' and '/help'
@bot.message_handler(commands=['help', 'start'])
def send_welcome(message):
	try:
		with conn.cursor() as cursor:
			sql = f"SELECT * FROM `users` WHERE `t_id` = {message.from_user.id}"
			cursor.execute(sql)
			result = cursor.fetchone()
		if result:
			with conn.cursor() as cursor:
				if result['uname']!=message.from_user.first_name:
					sql = f"UPDATE `users` SET `uname` = '{message.from_user.first_name}'"
					cursor.execute(sql)
				conn.commit()
				bot.reply_to(message, f"<code>Welcome back</code>")
		else:
			with conn.cursor() as cursor:
				sql = f"INSERT INTO `users` (t_id,uname) VALUES ({message.from_user.id},'{message.from_user.first_name}')"
				cursor.execute(sql)	
				conn.commit()
				bot.reply_to(message, f"<code>Welcome</code>")
		bot.reply_to(message, """ Use Learning Material command for preparation./learning_material\n
			Use Refresh command to refresh the bot /refresh\n
			Use cancel command to interrupt any process /cancel\n
			Use Add Command to add questions to the bot /add_question\n
			Use Take test command to attend the quiz /take_test""")
	except Exception as e:
		bot.reply_to(message, f"<code>start: {e}</code>")
		print(f"<code>ERROR: {e}</code>")


@bot.message_handler(commands=['learning_material'])
def learning_material(message):
	try:
		global conn
		reply_markup = InlineKeyboardMarkup()
		reply_markup.row_width = 2
		for i, val in enumerate(SUBJECTS):
			reply_markup.add(InlineKeyboardButton(val, callback_data=f"[SUBJECTS] {val}"))
		bot.reply_to(message, "SELECT QUESTIONS FROM THE FOLLOWING:", reply_markup=reply_markup)
	except Exception as e:
		bot.reply_to(message, f"<code>start: {e}</code>")
		print(f"<code>ERROR: {e}</code>")


@bot.message_handler(commands=['refresh','cancel'])
def refresh(message):
	global conn,states, ans, c_op, ques_id,result,arr,l
	states= None
	ans=''
	c_op=''
	ques_id,l=0,0
	result=''
	arr=[]
	with conn.cursor() as cursor:
		cursor.execute(f"DELETE FROM level WHERE t_id='{message.from_user.id}'")
		if message.text=='/refresh':
			bot.reply_to(message, f"<code>I'm ready now, how can i help you?</code>")
		elif message.text=='/cancel':
			send_welcome(message)
		states=None
		conn.commit()


@bot.message_handler(commands=['take_test'])
def take_test(message):
	try:
		global conn, states, ans, c_op, ques_id,result,arr,l
		with conn.cursor() as cursor:
			if states==None or (states in ['0','1','2','3','4','5','6','7','8','9']):
				if ans=="":
					sql = f"SELECT * FROM `questions`"
					cursor.execute(sql)
					result = cursor.fetchall()
					i=0
					#print(len(result))
					while len(result)!=i:
						arr.append(result[i]['ques_id'])
						#arr.sort()
						i+=1
						#print(i)
						#print(arr)
						#print(result)
				if ans!='':
					l=l+1
					status=0
					if ans==c_op:
						status=1
					sql = f"INSERT INTO result VALUES ({message.from_user.id},{ques_id},'{ans}',{status})"
					cursor.execute(sql)
				if len(arr)==0 or l==10:
					states="res"
					take_test(message)
				if states!="res":
					arr.sort()
					n=randrange(0,len(arr))
					num=arr[n]
					arr.remove(num)
					option1=result[num-1]['op1']
					option2=result[num-1]['op2']
					option3=result[num-1]['op3']
					option4=result[num-1]['op4']
					question=result[num-1]['question']
					ques_id=result[num-1]['ques_id']
					c_op=result[num-1]['c_op']
					markup = types.ReplyKeyboardMarkup(row_width=1,one_time_keyboard=True)
					opt1 = types.KeyboardButton(option1)
					opt2 = types.KeyboardButton(option2)
					opt3 = types.KeyboardButton(option3)
					opt4 = types.KeyboardButton(option4)
					markup.add(opt1, opt2, opt3, opt4)
					bot.reply_to(message, f"{question}", reply_markup=markup)
					states=str(l)
			elif states=='res':
				bot.reply_to(message, f"<code>You Have Successfully completed the quiz \n For taking test again click on</code> /take_test<code> \nfor help click on</code> /help")
				sql=f"SELECT * FROM result WHERE t_id='{message.from_user.id}'"
				cursor.execute(sql)	
				result=cursor.fetchall()
				score=0
				if result:
					for i in range(len(result)):
						score+=result[i]['status']
				bot.reply_to(message, f"<code>You scored: {score} out of {len(result)}</code>")
				#sql1=f"DELETE FROM level WHERE t_id={message.from_user.id}"
				#cursor.execute(sql1)
				sql=f"DELETE FROM result WHERE t_id={message.from_user.id}"
				cursor.execute(sql)
				states=None
			conn.commit()
	except Exception as e:
		#bot.reply_to(message, f"<code>Take Test error: {e}</code>")
		x=9#print(f"<code>Take test error: {e}</code>")



@bot.message_handler(commands=['add_question'])
def add_question(message):
	try:
		global conn, states
		with conn.cursor() as cursor:
			if states==None:
				bot.reply_to(message, f"<code>Enter the Question - </code>")
				states='add_ques'
			elif states=='add_ques':
				bot.reply_to(message, f"<code>Enter the First Option - </code>")
				states='add_opt1'
			elif states=='add_opt1':
				bot.reply_to(message, f"<code>Enter the second Option - </code>")
				states='add_opt2'
			elif states=='add_opt2':
				bot.reply_to(message, f"<code>Enter the third Option - </code>")
				states='add_opt3'
			elif states=='add_opt3':
				bot.reply_to(message, f"<code>Enter the Fourth Option - </code>")
				states='add_opt4'
			elif states=='add_opt4':
				bot.reply_to(message, f"<code>Enter the Correct Option - </code>")
				states='add_optc'
			elif states=='add_optc':
				bot.reply_to(message, f"<code>You Have Successfully added the question,\n\nFor adding more QUESTIONS click on - </code> /add_question")
				sql=f"SELECT message FROM level WHERE t_id='{message.from_user.id}'"
				cursor.execute(sql)	
				result=cursor.fetchall()
				ques,opt1,opt2,opt3,opt4,optc=result[0]['message'],result[1]['message'],result[2]['message'],result[3]['message'],result[4]['message'],result[5]['message']
				sql = f"INSERT INTO questions(question,op1,op2,op3,op4,c_op) VALUES ('{ques}','{opt1}','{opt2}','{opt3}','{opt4}','{optc}')"
				sql1=f"DELETE FROM level WHERE t_id={message.from_user.id}"
				cursor.execute(sql1)
				cursor.execute(sql)
				states=None
			conn.commit()
	except Exception as e:
		bot.reply_to(message, f"<code>add question: {e}</code>")
		print(f"<code>Add question error: {e}</code>")




@bot.callback_query_handler(func=lambda call: True)
def callback_query(call):
	try:
		if "[SUBJECTS]" in call.data:
			args = (call.data).split(" ")
			subject = args[1]
			if subject == SUBJECTS[0]:
				bot.answer_callback_query(call.id)
				bot.send_message(call.message.chat.id, f"<b>QUESTIONS LINK: </b>https://www.youtube.com/watch?v=PkMUHheXesk&t=32s")
			elif subject == SUBJECTS[1]:
				bot.answer_callback_query(call.id)
				bot.send_message(call.message.chat.id, f"<b>QUESTIONS LINK: </b>https://careerdost.in/aptitude-questions/age-problems")
			elif subject == SUBJECTS[2]:
				bot.answer_callback_query(call.id)
				bot.send_message(call.message.chat.id, f"<b>QUESTIONS LINK: </b>https://www.indiabix.com/aptitude/problems-on-trains/")
			elif subject == SUBJECTS[3]:
				bot.answer_callback_query(call.id)
				bot.send_message(call.message.chat.id, f"<b>QUESTIONS LINK: </b> https://prepinsta.com/learn-logical-reasoning/")
			elif subject == SUBJECTS[4]:
				bot.answer_callback_query(call.id)
				bot.send_message(call.message.chat.id, f"<b>QUESTIONS LINK: </b> https://prepinsta.com/top-100-codes/")
	except Exception as e:
		print("ERROR: "+e)


# Handle all other messages with content_type 'text' (content_types defaults to ['text'])
@bot.message_handler(func=lambda message: True)
def echo_message(message):
	global conn, states, ans
	with conn.cursor() as cursor:
		if states == None:
			bot.reply_to(message, f"This is not a valid request!!!\n\n please use /help command for user manual.")
		elif states == "add_ques" or states == "add_opt1" or states == "add_opt2" or states == "add_opt3" or states == "add_opt4" or states == "add_optc":
			sql = f"INSERT INTO level VALUES ({message.from_user.id},'{message.text}','{states}')"
			cursor.execute(sql)
			add_question(message)
		elif states in ['0','1','2','3','4','5','6','7','8','9']:
			ans=message.text
			take_test(message)
		elif states=="res":
			take_test(message)
		conn.commit()


bot.polling()
