
首先，定义了一个回调函数

handler - 这条消息所对应的通道号，the channel id this message 


using pi_msg_callback_t = int(int handler, p_pi_msg_envelope_t p_env, const char *body, unsigned int len);