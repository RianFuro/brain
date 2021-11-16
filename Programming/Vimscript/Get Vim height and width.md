Vim initializes the options `columns` and `lines` to the total width and height in characters it can display when starting.
You can set those options, but it will mess up the display if used in a terminal session.
In graphical clients, this can be used to resize the window, as vim will try to adjust the window frame to these settings.

See `:h columns` and `:h lines` respectively.

In lua, the equivalent setinngs can be accessed with `vim.o.columns` and `vim.o.lines`.