# awz00mbaranzeb00rd
### Very basic fork to make gr8w8upd8m8 compatible with current Python and modules.

## A Wii balance board API

This is based off [gr8w8upd8m8](https://github.com/skorokithakis/gr8w8upd8m8/) and it is made to support Python3, for people who are going to work with a Balance Board, just like me.

## Requirements

To run `gr8w8upd8m8`, you need:
* Linux (or Windows if you are willing to compile the bluetooth module)
* `pybluez` module
* Bluetooth.

## Pairing the board (permamently)

Thanks to Ryan Myers for the following:

Install `bluez bluez-utils python-bluez`, and run the included `xwiibind.sh`. Follow the prompts, and your balance
board should be paired by the end of this. Notice that BlueZ 4.99 is required, BlueZ 5+ changes the DBus API in
incompatible ways.

## Usage

You can run it with:

    python3 gr8w8upd8m8.py

It will prompt you to put the board in sync mode and it will search for and connect to it.

If you already know the address, you can just specify it:

    python3 gr8w8upd8m8.py [00:11:22:33:44:AB]

That will skip the discovery process, and connect directly.

`awz00mbaranzeb00rd` uses the `bluez-test-device` utility of `bluez-utils` to disconnect the board at the end, which causes
the board to shut off. 

Pairing it with the OS will allow you to use the front button to reconnect to it and run the
script. (Which this _might_ work for you, it did *not* work for me)

Calculating the final weight is done by calculating the mode of all the event data, rounded to one decimal digit.

Feel free to use processor.weight to do whatever you want with the calculated weight

## Credits

Thanks go to:

* [gr8w8upd8m8](https://github.com/skorokithakis/gr8w8upd8m8/), for providing the base script.

