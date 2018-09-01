---
layout:     post
title:      电报群批量踢人脚本
subtitle:   利用群内管理员bot批量踢人
date:       2018-07-05
author:     yandenghong
header-img: img/post-moon.png
catalog: true
tags:
    - Python
    - 脚本
    - 电报
    - Telegram
---

# 前言
此脚本基于python3.5，依赖requests库。安装就不赘述了。需要在同级目录下的一个电报id的txt文件（每个电报id占一行）。

另外，因为telegram被墙, 因此这个脚本需要在可科学上网的环境下执行。
# 正文
···python
    import sys
    import requests


    class Bot:

        def __init__(self, token):
            self.token = self._validate_token(token)
            self.base_url = 'https://api.telegram.org/bot' + self.token

        @staticmethod
        def _validate_token(token):
            url = '{0}/getMe'.format('https://api.telegram.org/bot' + token)
            result = requests.get(url, timeout=100)

            if not result.json()['ok']:
                raise ValueError('invalid bot token')
            return token

        @staticmethod
        def _get_telegram_ids_by_file(txt_name=sys.argv[1]):
            """
            Get a list of telegram ids from a file.

            Args:
                txt_name: (:obj:`str`) telegram_ids filename.Default is the first argument of the command line.

            Returns:
                (:obj:`list`) list of telegram ids

            """
            with open('./{}.txt'.format(txt_name), 'r') as f:
                return [user_id.strip("\n") for user_id in f if user_id != "\n"]

        def kick_chat_member(self, chat_id, user_id):
            """
            Use this method to kick a user from a group or a supergroup.

            Args:
                chat_id (:obj:`int` | :obj:`str`): Unique identifier for the target chat.
                user_id (:obj:`int`): Telegram id of the target member.

            Returns:
                :obj:`bool` On success, ``True`` is returned.

            """
            url = '{0}/kickChatMember'.format(self.base_url)

            data = {'chat_id': chat_id, 'user_id': user_id}

            result = requests.post(url, data, timeout=100)

            return result.json()

        def batch_kick_chat_member(self, chat_id):
            """Batch kicking members.

            Returns:
                actual number of kicked member.
            """
            actual_count = 0
            for telegram_id in self._get_telegram_ids_by_file():

                result = self.kick_chat_member(chat_id, telegram_id)
                if result['ok']:
                    # 在这里添加输出方便查看
                    print("成功踢出成员:{0}".format(telegram_id))
                    actual_count += 1
            # 在这里添加输出方便查看
            print("清理完毕, 本次清理了{0}个成员".format(actual_count))
            return actual_count


    if __name__ == '__main__':
        bot = Bot('Your Token')
        bot.batch_kick_chat_member('Your Chat id')
···
# 执行
__python3 你的脚本名 电报id的txt文件名(不包含.txt)__
