# ev-morse
I've never written code in C before (using LLM's to their full potential). :) <br>
The code allows for the binding of multiple commands to a single key. (single, double, long press, ... any morse combination)<br>
Uses root privileges because it reads output from /dev/input. Not sure how else to do it, this is probably some cybersecurity cardinal sin. <br>
¯\\_ (ツ) _/¯ <i>oh well</i>, So be sure to read the code, it's less than 100 lines.

# Usage
The number following event varies, check with `sudo evtest /dev/input/event0` and increasing the number gradually
```
gcc -o evmorse evmorse.c
sudo ./evmorse /dev/input/event0
```
The first line compiles the code, then the second line runs the program.

# Changing bindings
The key codes will vary from device to device.<br>
&emsp; *IMPORTANT!* &emsp; Keycodes differ between X11 and /dev &ensp; (Ex. For me numpad minus is 82 in xev, but 74 in evtest)<br>
Here is a general example of how the bindings would be set. <br>
```
    switch (keyc) {
        case 164:
            system("playerctl play-pause");
            break;
        case 163:
            for(int i = 0; i < len; i++) {
                system("");
            }
            break;
        case 165:
            if (strcmp(pattern, "01") == 0){
                system("playerctl next");
            } else if (strcmp(pattern, "00") == 0){
                system("playerctl position 10+");
            }
            break;
    }
```
`strcmp()` allows us to compare the pattern with a str. In this case 01 or 00, (short, long) and (short, short) presses respectively. <br>
`keyc` is the variable holding the key code value, to find the keycode use `sudo evtest /dev/input/event0`. (Make sure to set the correct event, look at Usage) <br>
`system()` allows us to run commands as if they were run from the terminal. It is under the assumption that UID is 1000. <br>
`for()` allows us to loop command(s) based on the length (`len` variable) of the pattern.

# Changing timings
The expiration timing from no key press is defined in `timer.it_value.tv_sec` and `timer.it_value.tv_usec`
The long and short press timing comes from how events are handled. A potential solution would be to change the threshold for hold to be higher. <br>
Here is an example solution (changed lines are marked with //):
```
            if (ev.value == 2) {                                //
                                                                //
                hold++;
            } else if (ev.value == 0 && hold > 19) {            //
                hold = 0;
                printf("\nKey code: %d, Value: %s ", ev.code, pattern);
                fflush(stdout);
            } else if (hold < 20 && ev.value == 0) {            //
                strcat(pattern,"0");
                printf("\nKey code: %d, Value: %s ", ev.code, pattern);
                fflush(stdout);
            } else if (ev.value == 1) {
                hold = 0;
            }
            if (hold == 20) {                                   //
                strcat(pattern,"1");                            //
            }                                                   //
            setitimer(ITIMER_REAL, &timer, NULL);
```
# Dependencies
gcc and gcc-libs ? I think, but am not sure.

# Quirks
- After opening the /dev/input the UID is changed to 1000 (default non root user). This could be changed if we wanted to run commands as a different user.
- The low level manner in which the keyboard is capture means that functions work when a key is disabled with xmodmap

# More binding examples
My audio control setup:
```
    switch (keyc) {
        case 164:
            system("playerctl --player=spotify,cmus,%%any play-pause");
            break;
        case 163:
            if (strcmp(pattern, "1") == 0) {
                system("playerctl --player=spotify,cmus,%%any next");
            } else if (len > 1) {
                for(int i = 1; i < len; i++) {
                    system("playerctl --player=spotify,cmus,%%any next");
                } 
            } else {
                system("playerctl --player=spotify,cmus,%%any position 10+");
            }
            break;
        case 165:
            if (strcmp(pattern, "1") == 0) {
                system("playerctl --player=spotify,cmus,%%any previous");
            } else if (len > 1) {
                for(int i = 1; i < len; i++) {
                    system("playerctl --player=spotify,cmus,%%any previous");
                } 
            } else {
                system("playerctl --player=spotify,cmus,%%any position 10-");
            }
            break;
        case 96:
            break;
        case 74:
            if (len == 1){
                system("playerctl --player=spotify volume 0.5");
            } else if (len == 2) {
                system("playerctl --player=spotify volume 0.7");
            } else if (len == 3) {
                system("playerctl --player=spotify volume 1");
            }
            break;
    }
```
Send left click to position (w/ xdotool): <br>
\* Uses regex to parse output of xdotool getmouselocation, which is why it's so bulky *
```
FILE* fp;
char mpos[50];
regex_t regex;
int reti;
const char *mregex = "x:([0-9]+) y:([0-9]+)";
size_t nmatch = 3;
regmatch_t pmatch[3];


// ^^ Place above the function ^^

        case 65:
            fp = popen("xdotool getmouselocation", "r");
            fgets(mpos, sizeof(mpos)-1, fp);
            reti = regcomp(&regex, mregex, REG_EXTENDED);
            reti = regexec(&regex, mpos, nmatch, pmatch, 0);
            pclose(fp);
            system("xdotool mousemove 1900 1050 && xdotool click 1");
            sprintf(pattern, "xdotool mousemove %.*s %.*s", (int)(pmatch[1].rm_eo - pmatch[1].rm_so), mpos + pmatch[1].rm_so, (int)(pmatch[2].rm_eo - pmatch[2].rm_so), mpos + pmatch[2].rm_so);
            system(pattern);
            regfree(&regex);
            break;
```
